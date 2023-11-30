using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Text;
using log4net;
using Resin.Analysis;
using Resin.IO;
using Resin.IO.Read;
using Resin.Querying;
using Resin.System;

namespace Resin
{
    public class Collector : IDisposable
    {
        private readonly string _directory;
        private static readonly ILog Log = LogManager.GetLogger(typeof(Collector));
        private readonly IxInfo _ix;
        private readonly IDictionary<Term, IList<DocumentPosting>> _termCache;
        private readonly Dictionary<string, PostingsReader> _postingReaders;
        private readonly IScoringScheme _scorer;
        private readonly IList<LcrsTreeReader> _trieReaders;

        public Collector(string directory, IxInfo ix, IScoringScheme scorer)
        {
            _directory = directory;
            _ix = ix;
            _termCache = new Dictionary<Term, IList<DocumentPosting>>();
            _postingReaders = new Dictionary<string, PostingsReader>();
            _scorer = scorer;
            _trieReaders = new List<LcrsTreeReader>();
        }

        public IEnumerable<DocumentScore> Collect(QueryContext query)
        {
            var mappingTime = Time();

            Scan(query);
            Score(query);
            Reduce(query);

            Log.DebugFormat("mapped {0} in {1}", query, mappingTime.Elapsed);

            return query.Resolve()
                .OrderByDescending(s => s.Score);
        }

        private void Reduce(QueryContext query)
        {
            query.Reduced = query.Scores.Aggregate(QueryContext.JoinOr);
        }

        private void Scan(QueryContext query)
        {
            if (query == null) throw new ArgumentNullException("query");

            var time = Time();

            if (query.Fuzzy)
            {
                query.Terms = GetTreeReader(query.Field).Near(query.Value, query.Edits)
                    .Select(token => new Term(query.Field, token));
            }
            else if (query.Prefix)
            {
                query.Terms = GetTreeReader(query.Field).StartsWith(query.Value)
                    .Select(token => new Term(query.Field, token));
            }
            else
            {
                if (GetTreeReader(query.Field).HasWord(query.Value))
                {
                    query.Terms = new List<Term> { new Term(query.Field, query.Value) };
                }
                else
                {
                    query.Terms = new List<Term>();
                }
            }

            foreach (var child in query.Children)
            {
                Scan(child);
            }

            Log.DebugFormat("scanned {0} in {1}", query, time.Elapsed);
        }

        private void Score(QueryContext query)
        {
            var time = Time();

            query.Scores = (from term in query.Terms 
                          let docsInCorpus = _ix.DocumentCount.DocCount[term.Field] let postings = GetPostings(term) 
                          select Score(postings, docsInCorpus));

            foreach (var child in query.Children)
            {
                Score(child);
            }

            Log.DebugFormat("scored {0} in {1}", query, time.Elapsed);
        }

        private IEnumerable<DocumentScore> Score(IList<DocumentPosting> postings, int docsInCorpus)
        {
            var scorer = _scorer.CreateScorer(docsInCorpus, postings.Count);

            foreach (var posting in postings)
            {
                var hit = new DocumentScore(posting.DocumentId, posting.Count);

                scorer.Score(hit);

                yield return hit;
            }
        }
        
        private IList<DocumentPosting> GetPostings(Term term)
        {
            IList<DocumentPosting> postings;

            if (!_termCache.TryGetValue(term, out postings))
            {
                var time = Time();

                var fileId = term.ToPostingsFileId();
                var fileName = Path.Combine(_directory, fileId + ".pos");
                PostingsReader reader;

                if (!_postingReaders.TryGetValue(fileId, out reader))
                {
                    if (!_postingReaders.TryGetValue(fileId, out reader))
                    {
                        var fs = File.Open(fileName, FileMode.Open, FileAccess.Read, FileShare.Read);
                        var sr = new StreamReader(fs, Encoding.ASCII);

                        reader = new PostingsReader(sr);

                        _postingReaders.Add(fileId, reader);
                    }
                }

                postings = reader.Read(term).ToList();
                _termCache.Add(term, postings);

                Log.DebugFormat("read {0} postings from {1} in {2}", postings.Count(), fileName, time.Elapsed);
            }

            return postings;
        }

        private LcrsTreeReader GetTreeReader(string field)
        {
            var fileName = Path.Combine(_directory, field.ToTrieFileId() + ".tri");
            var fs = File.Open(fileName, FileMode.Open, FileAccess.Read, FileShare.Read);
            var sr = new StreamReader(fs, Encoding.Unicode);
            var reader = new LcrsTreeReader(sr);

            _trieReaders.Add(reader);
            
            return reader;
        }

        private Stopwatch Time()
        {
            var timer = new Stopwatch();
            timer.Start();
            return timer;
        }

        public void Dispose()
        {
            foreach (var dr in _postingReaders.Values)
            {
                dr.Dispose();
            }
            foreach (var r in _trieReaders)
            {
                r.Dispose();
            }
        }
    }
}