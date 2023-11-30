using System;
using System.Diagnostics;
using System.Globalization;
using System.IO;
using System.Linq;
using System.Text;
using log4net;
using Resin.Analysis;
using Resin.IO;
using Resin.IO.Read;
using Resin.Querying;
using Resin.Sys;

namespace Resin
{
    public class Searcher : IDisposable
    {
        private static readonly ILog Log = LogManager.GetLogger(typeof(Searcher));
        private readonly string _directory;
        private readonly QueryParser _parser;
        private readonly IScoringScheme _scorer;
        private readonly IxInfo _ix;

        public Searcher(string directory, QueryParser parser, IScoringScheme scorer)
        {
            _directory = directory;
            _parser = parser;
            _scorer = scorer;
            _ix = IxInfo.Load(Path.Combine(_directory, "0.ix"));
        }

        public Result Search(string query, int page = 0, int size = 10000, bool returnTrace = false)
        {
            using (var collector = new Collector(_directory, _ix, _scorer))
            {
                var time = Time();

                var q = _parser.Parse(query);

                if (q == null)
                {
                    return new Result { Docs = Enumerable.Empty<Document>() };
                }

                Log.DebugFormat("parsed query {0} in {1}", q, time.Elapsed);

                var scored = collector.Collect(q).ToList();
                var skip = page * size;
                var paged = scored.Skip(skip).Take(size).ToDictionary(x => x.DocId, x => x);
                var docs = paged.Values.Select(GetDoc);

                return new Result { Docs = docs, Total = scored.Count }; 
            }
        }

        private Document GetDoc(DocumentScore score)
        {
            var fileId = score.DocId.ToDocFileId();
            var fileName = Path.Combine(_directory, fileId + ".doc");
            var fs = File.Open(fileName, FileMode.Open, FileAccess.Read, FileShare.Read);
            var sr = new StreamReader(fs, Encoding.ASCII);

            using (var reader = new DocumentReader(sr))
            {
                var doc = reader.Get(score.DocId);

                doc.Fields["__score"] = score.Score.ToString(CultureInfo.InvariantCulture);

                return doc;    
            }
        }

        private static Stopwatch Time()
        {
            var timer = new Stopwatch();
            timer.Start();
            return timer;
        }

        public void Dispose()
        {
        }
    }
}