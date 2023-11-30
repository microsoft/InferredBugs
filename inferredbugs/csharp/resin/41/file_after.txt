using System;
using System.Collections.Concurrent;
using System.Diagnostics;
using System.IO;

namespace Sir.Store
{
    /// <summary>
    /// Indexing session targeting a single collection.
    /// </summary>
    public class TermIndexSession : CollectionSession, IDisposable, ILogger
    {
        private readonly IConfigurationProvider _config;
        private readonly IStringModel _model;
        private readonly ConcurrentDictionary<long, VectorNode> _dirty;
        private Stream _postingsStream;
        private readonly Stream _vectorStream;

        public TermIndexSession(
            ulong collectionId,
            SessionFactory sessionFactory, 
            IStringModel model,
            IConfigurationProvider config) : base(collectionId, sessionFactory)
        {
            _config = config;
            _model = model;
            _dirty = new ConcurrentDictionary<long, VectorNode>();
            _postingsStream = SessionFactory.CreateAppendStream(Path.Combine(SessionFactory.Dir, $"{CollectionId}.pos"));
            _vectorStream = SessionFactory.CreateAppendStream(Path.Combine(SessionFactory.Dir, $"{CollectionId}.vec"));
        }

        public void Put(long docId, long keyId, string value)
        {
            var ix = GetOrCreateIndex(keyId);
            var tokens = _model.Tokenize(value);

            foreach (var vector in tokens.Embeddings)
            {
                GraphBuilder.Add(ix, new VectorNode(vector, docId), _model);
            }
        }

        private void Validate((long keyId, long docId, AnalyzedData tokens) item)
        {
            var tree = GetOrCreateIndex(item.keyId);

            foreach (var vector in item.tokens.Embeddings)
            {
                var hit = PathFinder.ClosestMatch(tree, vector, _model);

                if (hit.Score < _model.IdenticalAngle)
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

        private VectorNode GetOrCreateIndex(long keyId)
        {
            return _dirty.GetOrAdd(keyId, new VectorNode());
        }

        public void CreatePage()
        {
            var time = Stopwatch.StartNew();

            foreach (var column in _dirty)
            {
                using (var serializer = new ColumnWriter(CollectionId, column.Key, SessionFactory))
                {
                    serializer.CreatePage(column.Value, _vectorStream, _postingsStream, _model);
                }
            };

            _dirty.Clear();

            SessionFactory.ClearPageInfo();

            this.Log(string.Format($"serialized {_dirty.Count} pages in {time.Elapsed}"));
        }

        public void Dispose()
        {
            _postingsStream.Dispose();
            _vectorStream.Dispose();
        }
    }
}