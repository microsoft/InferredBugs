using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using log4net;
using Resin.IO;

namespace Resin
{
    /// <summary>
    /// A reader that provides thread-safe access to an index. Not lock-free.
    /// </summary>
    public class Searcher : IDisposable
    {
        private static readonly ILog Log = LogManager.GetLogger(typeof(Searcher));
        private readonly string _directory;
        private readonly QueryParser _parser;
        private readonly IScoringScheme _scorer;
        private readonly IndexInfo _ix;
        private readonly Dictionary<string, DocumentFile> _docContainers;
        private static readonly object Sync = new object();

        public Searcher(string directory, QueryParser parser, IScoringScheme scorer)
        {
            _directory = directory;
            _parser = parser;
            _scorer = scorer;
            _docContainers = new Dictionary<string, DocumentFile>();

            _ix = IndexInfo.Load(Path.Combine(_directory, "0.ix"));
        }

        public Result Search(string query, int page = 0, int size = 10000, bool returnTrace = false)
        {
            var timer = new Stopwatch();
            timer.Start();
            var collector = new Collector(_directory, _ix);
            var q = _parser.Parse(query);
            if (q == null)
            {
                return new Result { Docs = Enumerable.Empty<IDictionary<string, string>>() };
            }
            Log.DebugFormat("parsed query {0} in {1}", q, timer.Elapsed);
            var scored = collector.Collect(q, page, size, _scorer).ToList();
            var skip = page * size;
            var paged = scored.Skip(skip).Take(size).ToDictionary(x => x.DocId, x => x);
            var docs = paged.Values.Select(s => GetDoc(s.DocId));
            return new Result { Docs = docs, Total = scored.Count };  
        }


        private IDictionary<string, string> GetDoc(string docId)
        {
            var containerId = docId.ToDocContainerId();
            DocumentFile container;
            if (!_docContainers.TryGetValue(containerId, out container))
            {
                lock (Sync)
                {
                    if (!_docContainers.TryGetValue(containerId, out container))
                    {
                        container = new DocumentFile(_directory, containerId);
                        _docContainers.Add(containerId, container);
                    }
                }
            }
            return container.Get(docId).Fields;
        }

        public void Dispose()
        {
            foreach (var dc in _docContainers.Values)
            {
                dc.Dispose();
            }
        }
    }
}