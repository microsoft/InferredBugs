using System;
using System.Collections;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Threading.Tasks;

namespace Sir.Store
{
    /// <summary>
    /// Read session targeting a single collection.
    /// </summary>
    public class ReadSession : CollectionSession
    {
        private readonly DocIndexReader _docIx;
        private readonly DocReader _docs;
        private readonly ValueIndexReader _keyIx;
        private readonly ValueIndexReader _valIx;
        private readonly ValueReader _keyReader;
        private readonly ValueReader _valReader;
        private readonly RemotePostingsReader _postingsReader;
        private readonly StreamWriter _log;
        private readonly IList<Stream> _streams;

        public ReadSession(string collectionId, 
            SessionFactory sessionFactory, 
            IConfigurationProvider config) 
            : base(collectionId, sessionFactory)
        {
            var collection = collectionId.ToHash();

            ValueStream = sessionFactory.CreateAsyncReadStream(Path.Combine(sessionFactory.Dir, string.Format("{0}.val", collection)));
            KeyStream = sessionFactory.CreateAsyncReadStream(Path.Combine(sessionFactory.Dir, string.Format("{0}.key", collection)));
            DocStream = sessionFactory.CreateAsyncReadStream(Path.Combine(sessionFactory.Dir, string.Format("{0}.docs", collection)));
            ValueIndexStream = sessionFactory.CreateAsyncReadStream(Path.Combine(sessionFactory.Dir, string.Format("{0}.vix", collection)));
            KeyIndexStream = sessionFactory.CreateAsyncReadStream(Path.Combine(sessionFactory.Dir, string.Format("{0}.kix", collection)));
            DocIndexStream = sessionFactory.CreateAsyncReadStream(Path.Combine(sessionFactory.Dir, string.Format("{0}.dix", collection)));

            _docIx = new DocIndexReader(DocIndexStream);
            _docs = new DocReader(DocStream);
            _keyIx = new ValueIndexReader(KeyIndexStream);
            _valIx = new ValueIndexReader(ValueIndexStream);
            _keyReader = new ValueReader(KeyStream);
            _valReader = new ValueReader(ValueStream);
            _postingsReader = new RemotePostingsReader(config);
            _streams = new List<Stream>();
            _log = Logging.CreateWriter("readsession");
        }

        public async Task<ReadResult> Read(Query query, int take)
        {
            IDictionary<ulong, float> result = await Reduce(query);

            if (result == null)
            {
                _log.Log("found nothing for query {0}", query);

                return new ReadResult { Total = 0, Docs = new IDictionary[0] };
            }
            else
            {
                IEnumerable<KeyValuePair<ulong, float>> sorted = result.OrderByDescending(x => x.Value);
                
                if (take > 0)
                    sorted = sorted.Take(take);

                var docs = await ReadDocs(sorted);

                return new ReadResult { Total = result.Count, Docs = docs };
            }
        }

        public async Task<ReadResult> Read(Query query)
        {
            var timer = new Stopwatch();

            // Get doc IDs and their score
            IDictionary<ulong, float> docIds = await Reduce(query);

            if (docIds == null)
            {
                _log.Log("found nothing for query {0}", query);

                return new ReadResult { Total = 0, Docs = new IDictionary[0] };
            }
            else
            {
                if (docIds.Count < 101)
                {
                    return new ReadResult { Total = docIds.Count, Docs = await ReadDocs(docIds) };
                }

                timer.Restart();

                var sorted = new List<KeyValuePair<ulong, float>>();
                var ordered = docIds.OrderByDescending(x => x.Value);
                float topScore = 0;
                int index = 0;

                foreach (var s in ordered)
                {
                    if (index++ == 0)
                    {
                        topScore = s.Value;
                        sorted.Add(s);
                    }
                    else if (s.Value == topScore)
                    {
                        sorted.Add(s);
                    }
                    else
                    {
                        break;
                    }
                }

                _log.Log("sorted and reduced {0} postings for query {1} in {2}",
                    docIds.Count, query, timer.Elapsed);

                var docs = await ReadDocs(sorted);

                _log.Log("read {0} documents from disk for query {1} in {2}",
                    docs.Count, query, timer.Elapsed);

                return new ReadResult { Total = docIds.Count, Docs = docs };
            }
        }

        private NodeReader GetIndexReader(long keyId)
        {
            var cid = CollectionId.ToHash();
            var ixFileName = Path.Combine(SessionFactory.Dir, string.Format("{0}.{1}.ix", cid, keyId));
            var pageIxFileName = Path.Combine(SessionFactory.Dir, string.Format("{0}.{1}.ixp", cid, keyId));
            var vecFileName = Path.Combine(SessionFactory.Dir, string.Format("{0}.vec", cid));

            var ixStream = SessionFactory.CreateAsyncReadStream(ixFileName);
            var vecStream = SessionFactory.CreateAsyncReadStream(vecFileName);
            var ixpStream = SessionFactory.CreateAsyncReadStream(pageIxFileName);

            _streams.Add(ixStream);
            _streams.Add(vecStream);
            _streams.Add(ixpStream);

            return new NodeReader(ixStream, vecStream, ixpStream);
        }

        public NodeReader GetIndexReader(ulong keyHash)
        {
            long keyId;
            if (!SessionFactory.TryGetKeyId(keyHash, out keyId))
            {
                return null;
            }

            return GetIndexReader(keyId);
        }

        private async Task<IDictionary<ulong, float>> Reduce(Query query)
        {
            try
            {
                IDictionary<ulong, float> result = null;

                var cursor = query;

                while (cursor != null)
                {
                    var timer = new Stopwatch();
                    timer.Start();

                    var keyHash = cursor.Term.Key.ToString().ToHash();
                    IList<Hit> matching = null;

                    using (var indexReader = GetIndexReader(keyHash))
                    {
                        if (indexReader != null)
                        {
                            var termVector = cursor.Term.Value.ToString().ToCharVector();

                            matching = indexReader.ClosestMatch(termVector);
                        }
                    }

                    _log.Log("scan found {0} matches in {1}", matching.Count, timer.Elapsed);

                    timer.Restart();

                    var cutoff = matching[0].Score * 0.85f;
                    var qualifiedMatches = matching.GroupBy(x => x.Score).Where(g => g.Key >= cutoff).SelectMany(g => g).ToList();
                    var docIds = new Dictionary<ulong, float>();

                    foreach (var match in qualifiedMatches)
                    {
                        foreach (var id in await _postingsReader.Read(CollectionId, match.PostingsOffset))
                        {
                            if (docIds.ContainsKey(id))
                            {
                                docIds[id] += match.Score;
                            }
                            else
                            {
                                docIds.Add(id, match.Score);
                            }
                        }
                    }

                    _log.Log("fetched {0} doc IDs in {1}", docIds.Count, timer.Elapsed);

                    timer.Restart();

                    if (result == null)
                    {
                        result = docIds;
                    }
                    else
                    {
                        if (cursor.And)
                        {
                            var aggregatedResult = new Dictionary<ulong, float>();

                            foreach (var doc in result)
                            {
                                float score;

                                if (docIds.TryGetValue(doc.Key, out score))
                                {
                                    aggregatedResult[doc.Key] = score + doc.Value;
                                }
                            }

                            result = aggregatedResult;
                        }
                        else if (cursor.Not)
                        {
                            foreach (var id in docIds.Keys)
                            {
                                result.Remove(id);
                            }
                        }
                        else // Or
                        {
                            foreach (var id in docIds)
                            {
                                float score;

                                if (result.TryGetValue(id.Key, out score))
                                {
                                    result[id.Key] = score + id.Value;
                                }
                                else
                                {
                                    result.Add(id.Key, id.Value);
                                }
                            }
                        }
                    }

                    _log.Log("reduced {0} to {1} docs in {2}",
                        cursor, result.Count, timer.Elapsed);

                    cursor = cursor.Next;
                }

                return result;
            }
            catch (Exception ex)
            {
                _log.Log(ex);
                throw;
            }
        }

        private async Task<IList<IDictionary>> ReadDocs(IEnumerable<KeyValuePair<ulong, float>> docs)
        {
            var timer = new Stopwatch();
            timer.Start();

            var result = new List<IDictionary>();

            foreach (var d in docs)
            {
                var docInfo = await _docIx.ReadAsync(d.Key);

                if (docInfo.offset < 0)
                {
                    continue;
                }

                var docMap = await _docs.Read(docInfo.offset, docInfo.length);
                var doc = new Dictionary<IComparable, IComparable>();

                for (int i = 0; i < docMap.Count; i++)
                {
                    var kvp = docMap[i];
                    var kInfo = await _keyIx.ReadAsync(kvp.keyId);
                    var vInfo = await _valIx.ReadAsync(kvp.valId);
                    var key = await _keyReader.ReadAsync(kInfo.offset, kInfo.len, kInfo.dataType);
                    var val = await _valReader.ReadAsync(vInfo.offset, vInfo.len, vInfo.dataType);

                    doc[key] = val;
                }

                doc["__docid"] = d.Key;
                doc["__score"] = d.Value;

                result.Add(doc);
            }

            _log.Log("read {0} docs in {1}", result.Count, timer.Elapsed);

            return result;
        }

        public override void Dispose()
        {
            base.Dispose();

            foreach (var stream in _streams)
            {
                stream.Dispose();
            }
        }
    }

    public class ReadResult
    {
        public long Total { get; set; }
        public IList<IDictionary> Docs { get; set; }
    }
}
