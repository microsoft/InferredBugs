using Sir.Core;
using System;
using System.Collections;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;

namespace Sir.Store
{
    /// <summary>
    /// Indexing session targeting a single collection.
    /// </summary>
    public class TermIndexSession : CollectionSession, IDisposable, ILogger
    {
        private readonly IConfigurationProvider _config;
        private readonly ITokenizer _tokenizer;
        private readonly IDictionary<long, VectorNode> _dirty;
        private readonly Stream _vectorStream;
        private bool _flushed;
        private bool _flushing;
        private readonly ProducerConsumerQueue<(long docId, long keyId, AnalyzedString tokens)> _modelBuilder;
        private readonly ConcurrentDictionary<long, NodeReader> _indexReaders;
        private readonly long[] _excludeKeyIds;

        public TermIndexSession(
            string collectionName,
            ulong collectionId,
            SessionFactory sessionFactory, 
            ITokenizer tokenizer,
            IConfigurationProvider config,
            ConcurrentDictionary<long, NodeReader> indexReaders,
            params long[] excludeKeyIds) : base(collectionName, collectionId, sessionFactory)
        {
            _config = config;
            _tokenizer = tokenizer;
            _dirty = new ConcurrentDictionary<long, VectorNode>();
            _vectorStream = SessionFactory.CreateAppendStream(Path.Combine(SessionFactory.Dir, string.Format("{0}.vec", CollectionId)));

            var numThreads = int.Parse(_config.Get("write_thread_count"));

            _modelBuilder = new ProducerConsumerQueue<(long docId, long keyId, AnalyzedString tokens)>(
                numThreads, BuildModel);

            _indexReaders = indexReaders;

            _excludeKeyIds = excludeKeyIds;
        }

        /// <summary>
        /// Fields prefixed with "___" or "__" will not be indexed.
        /// Fields prefixed with "_" will not be tokenized.
        /// </summary>
        public void Index(IDictionary document)
        {
            var docId = (long)document["___docid"];

            foreach (var obj in document.Keys)
            {
                var key = (string)obj;
                AnalyzedString tokens = null;

                if (!key.StartsWith("__"))
                {
                    var keyHash = key.ToHash();
                    var keyId = SessionFactory.GetKeyId(CollectionId, keyHash);

                    if (_excludeKeyIds.Contains(keyId))
                    {
                        continue;
                    }

                    var val = document[key];
                    var str = val as string;

                    if (str == null || key[0] == '_')
                    {
                        var v = val.ToString();

                        if (!string.IsNullOrWhiteSpace(v))
                        {
                            tokens = new AnalyzedString
                            {
                                Original = v,
                                Source = v.ToCharArray(),
                                Tokens = new List<(int, int)> { (0, v.Length) }
                            };
                        }
                    }
                    else
                    {
                        tokens = _tokenizer.Tokenize(str);
                    }

                    if (tokens != null)
                    {
                        _modelBuilder.Enqueue((docId, keyId, tokens));
                    }
                }
            }

            this.Log("analyzed document {0} ", docId);
        }

        private void BuildModel((long docId, long keyId, AnalyzedString tokens) item)
        {
            var ix = GetOrCreateIndex(item.keyId);

            foreach (var vector in item.tokens.Embeddings())
            {
                ix.Add(new VectorNode(vector, item.docId), CosineSimilarity.Term, _vectorStream);
            }
        }

        public void Flush()
        {
            if (_flushing || _flushed)
                return;

            _flushing = true;

            this.Log("waiting for model builder");

            using (_modelBuilder)
            {
                _modelBuilder.Join();
            }

            using (_vectorStream)
            {
                _vectorStream.Flush();
                _vectorStream.Close();
            }

            var tasks = new Task[_dirty.Count];
            var taskId = 0;
            var columnWriters = new List<ColumnSerializer>();

            foreach (var column in _dirty)
            {
                var columnWriter = new ColumnSerializer(
                    CollectionId, column.Key, SessionFactory, new RemotePostingsWriter(_config, CollectionName));

                columnWriters.Add(columnWriter);

                tasks[taskId++] = columnWriter.SerializeColumnSegment(column.Value);
            }

            Task.WaitAll(tasks);

            _flushed = true;
            _flushing = false;

            this.Log(string.Format("***FLUSHED***"));
        }

        private void Validate((long keyId, long docId, AnalyzedString tokens) item)
        {
            if (item.keyId == 4 || item.keyId == 5)
            {
                var tree = GetOrCreateIndex(item.keyId);

                foreach (var vector in item.tokens.Embeddings())
                {
                    var hit = tree.ClosestMatch(vector, CosineSimilarity.Term.foldAngle);

                    if (hit.Score < CosineSimilarity.Term.identicalAngle)
                    {
                        throw new DataMisalignedException();
                    }

                    var valid = false;

                    foreach (var id in hit.Node.DocIds)
                    {
                        if (id == item.docId)
                        {
                            valid = true;
                            break;
                        }
                    }

                    if (!valid)
                    {
                        throw new DataMisalignedException();
                    }
                }
            }
        }

        private static readonly object _syncIndexAccess = new object();

        private VectorNode GetOrCreateIndex(long keyId)
        {
            VectorNode root;

            if (!_dirty.TryGetValue(keyId, out root))
            {
                lock (_syncIndexAccess)
                {
                    if (!_dirty.TryGetValue(keyId, out root))
                    {
                        root = new VectorNode();
                        _dirty.Add(keyId, root);
                    }
                }
            }

            return root;
        }

        public void Dispose()
        {
            Flush();
        }
    }
}