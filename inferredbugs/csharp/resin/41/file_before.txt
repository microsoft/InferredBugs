using System;
using System.Collections.Generic;
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
        private readonly IDictionary<long, VectorNode> _dirty;
        private readonly IDictionary<long, ColumnWriter> _writers;
        private Stream _postingsStream;
        private readonly Stream _vectorStream;

        public TermIndexSession(
            string collectionName,
            ulong collectionId,
            SessionFactory sessionFactory, 
            IStringModel model,
            IConfigurationProvider config,
            IDictionary<long, ColumnWriter> columnWriters) : base(collectionName, collectionId, sessionFactory)
        {
            _config = config;
            _model = model;
            _dirty = new Dictionary<long, VectorNode>();
            _postingsStream = SessionFactory.CreateAppendStream(Path.Combine(SessionFactory.Dir, $"{CollectionId}.pos"));
            _vectorStream = SessionFactory.CreateAppendStream(Path.Combine(SessionFactory.Dir, $"{CollectionId}.vec"));
            _writers = columnWriters;
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
            VectorNode node;

            if (!_dirty.TryGetValue(keyId, out node))
            {
                node = new VectorNode();
                _dirty.Add(keyId, node);
            }

            return node;
        }

        public void CreatePage()
        {
            var time = Stopwatch.StartNew();

            foreach (var column in _dirty)
            {
                ColumnWriter serializer;

                if (!_writers.TryGetValue(column.Key, out serializer))
                {
                    serializer = new ColumnWriter(CollectionId, column.Key, SessionFactory);
                    _writers.Add(column.Key, serializer);
                }

                serializer.CreatePage(column.Value, _vectorStream, _postingsStream, _model);
            };

            this.Log(string.Format($"serialized {_dirty.Count} pages in {time.Elapsed}"));

            _dirty.Clear();
        }

        public void Dispose()
        {
            _postingsStream.Dispose();
            _vectorStream.Dispose();
        }
    }
}