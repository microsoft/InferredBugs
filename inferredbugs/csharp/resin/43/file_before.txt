using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.IO.MemoryMappedFiles;
using System.Threading;

namespace Sir.Store
{
    /// <summary>
    /// Dispatcher of sessions.
    /// </summary>
    public class SessionFactory : IDisposable, ILogger
    {
        private ConcurrentDictionary<ulong, ConcurrentDictionary<ulong, long>> _keys;
        private readonly ConcurrentDictionary<string, IList<(long offset, long length)>> _pageInfo;
        private readonly Semaphore WriteSync = new Semaphore(1, 2);
        private readonly ConcurrentDictionary<string, MemoryMappedFile> _mmfs;

        public string Dir { get; }
        public IConfigurationProvider Config { get; }
        public IStringModel Model { get; }

        public SessionFactory(IConfigurationProvider config, IStringModel model)
        {
            var time = Stopwatch.StartNew();

            Dir = config.Get("data_dir");
            Config = config;

            if (!Directory.Exists(Dir))
            {
                Directory.CreateDirectory(Dir);
            }

            _keys = LoadKeys();
            _pageInfo = new ConcurrentDictionary<string, IList<(long offset, long length)>>();
            _mmfs = new ConcurrentDictionary<string, MemoryMappedFile>();
            Model = model;

            this.Log($"initiated in {time.Elapsed}");
        }



        public MemoryMappedFile OpenMMF(string fileName)
        {
            var mapName = fileName.Replace(":", "").Replace("\\", "_");

            try
            {
                return _mmfs.GetOrAdd(mapName, x =>
                {
                    return MemoryMappedFile.CreateFromFile(fileName, FileMode.Open, mapName, 0, MemoryMappedFileAccess.ReadWrite);
                });
            }
            catch
            {
                return _mmfs.GetOrAdd(mapName, x =>
                {
                    return MemoryMappedFile.OpenExisting(mapName);
                });
            }
        }

        public long GetDocCount(string collection)
        {
            var fileName = Path.Combine(Dir, $"{collection.ToHash()}.dix");

            if (!File.Exists(fileName))
                return 0;

            return new FileInfo(fileName).Length / (sizeof(long) + sizeof(int));
        }

        public void Truncate(ulong collectionId)
        {
            foreach (var file in Directory.GetFiles(Dir, $"{collectionId}*"))
                File.Delete(file);

            _pageInfo.Clear();

            _keys.Clear();

            this.Log($"truncated {collectionId}");
        }

        public void TruncateIndex(ulong collectionId)
        {
            foreach (var file in Directory.GetFiles(Dir, $"{collectionId}*.ix"))
            {
                File.Delete(file);
            }
            foreach (var file in Directory.GetFiles(Dir, $"{collectionId}*.ixp"))
            {
                File.Delete(file);
            }
            foreach (var file in Directory.GetFiles(Dir, $"{collectionId}*.vec"))
            {
                File.Delete(file);
            }
            foreach (var file in Directory.GetFiles(Dir, $"{collectionId}*.pos"))
            {
                File.Delete(file);
            }

            _pageInfo.Clear();
        }

        public void Write(Job job)
        {
            WriteSync.WaitOne();

            var timer = Stopwatch.StartNew();

            using (var writeSession = CreateWriteSession(job.CollectionId, job.Model))
            {
                writeSession.Write(job.Documents);
            }

            this.Log("executed {0} write in {1}", job.CollectionId, timer.Elapsed);

            WriteSync.Release();
        }

        public void ClearPageInfo()
        {
            _pageInfo.Clear();
        }

        public IList<(long offset, long length)> GetAllPages(string pageFileName)
        {
            return _pageInfo.GetOrAdd(pageFileName, key =>
            {
                using (var ixpStream = CreateReadStream(key))
                {
                    return new PageIndexReader(ixpStream).GetAll();
                }
            });
        }

        public void RefreshKeys()
        {
            _keys = LoadKeys();
        }

        public ConcurrentDictionary<ulong, ConcurrentDictionary<ulong, long>> LoadKeys()
        {
            var timer = Stopwatch.StartNew();
            var allkeys = new ConcurrentDictionary<ulong, ConcurrentDictionary<ulong, long>>();

            foreach (var keyFile in Directory.GetFiles(Dir, "*.kmap"))
            {
                var collectionId = ulong.Parse(Path.GetFileNameWithoutExtension(keyFile));
                ConcurrentDictionary<ulong, long> keys;

                if (!allkeys.TryGetValue(collectionId, out keys))
                {
                    keys = new ConcurrentDictionary<ulong, long>();
                    allkeys.GetOrAdd(collectionId, keys);
                }

                using (var stream = new FileStream(keyFile, FileMode.OpenOrCreate, FileAccess.Read, FileShare.ReadWrite))
                {
                    long i = 0;
                    var buf = new byte[sizeof(ulong)];
                    var read = stream.Read(buf, 0, buf.Length);

                    while (read > 0)
                    {
                        keys.GetOrAdd(BitConverter.ToUInt64(buf, 0), i++);

                        read = stream.Read(buf, 0, buf.Length);
                    }
                }
            }

            this.Log("loaded keys into memory in {0}", timer.Elapsed);

            return allkeys;
        }

        public void PersistKeyMapping(ulong collectionId, ulong keyHash, long keyId)
        {
            var fileName = Path.Combine(Dir, string.Format("{0}.kmap", collectionId));
            ConcurrentDictionary<ulong, long> keys;

            if (!_keys.TryGetValue(collectionId, out keys))
            {
                keys = new ConcurrentDictionary<ulong, long>();
                _keys.GetOrAdd(collectionId, keys);
            }

            if (!keys.ContainsKey(keyHash))
            {
                keys.GetOrAdd(keyHash, keyId);

                using (var stream = CreateAppendStream(fileName))
                {
                    stream.Write(BitConverter.GetBytes(keyHash), 0, sizeof(ulong));
                }
            }
        }

        public long GetKeyId(ulong collectionId, ulong keyHash)
        {
            return _keys[collectionId][keyHash];
        }

        public bool TryGetKeyId(ulong collectionId, ulong keyHash, out long keyId)
        {
            var keys = _keys.GetOrAdd(collectionId, new ConcurrentDictionary<ulong, long>());

            if (!keys.TryGetValue(keyHash, out keyId))
            {
                keyId = -1;
                return false;
            }
            return true;
        }

        public DocumentStreamSession CreateDocumentStreamSession(ulong collectionId)
        {
            return new DocumentStreamSession(collectionId, this, new DocumentReader(collectionId, this));
        }

        public WriteSession CreateWriteSession(ulong collectionId, IStringModel model)
        {
            var documentWriter = new DocumentWriter(collectionId, this);

            return new WriteSession(
                collectionId, 
                this, 
                documentWriter, 
                Config, 
                model, 
                CreateIndexSession(collectionId)
            );
        }

        public IndexSession CreateIndexSession(ulong collectionId)
        {
            return new IndexSession(collectionId, this, Model, Config);
        }

        public ReadSession CreateReadSession(ulong collectionId)
        {
            return new ReadSession(
                collectionId, this, Config, Model, new DocumentReader(collectionId, this));
        }

        public Stream CreateAsyncReadStream(string fileName, int bufferSize = 4096)
        {
            return File.Exists(fileName)
            ? new FileStream(fileName, FileMode.Open, FileAccess.Read, FileShare.ReadWrite, bufferSize, FileOptions.Asynchronous)
            : null;
        }

        public Stream CreateReadStream(string fileName, int bufferSize = 4096, FileOptions fileOptions = FileOptions.RandomAccess)
        {
            return File.Exists(fileName)
                ? new FileStream(fileName, FileMode.Open, FileAccess.Read, FileShare.ReadWrite, bufferSize, fileOptions)
                : null;
        }

        public Stream CreateAsyncAppendStream(string fileName, int bufferSize = 4096)
        {
            return new FileStream(fileName, FileMode.Append, FileAccess.Write, FileShare.ReadWrite, bufferSize, FileOptions.Asynchronous);
        }

        public Stream CreateAppendStream(string fileName, int bufferSize = 4096)
        {
            if (!File.Exists(fileName))
            {
                using (var fs = new FileStream(fileName, FileMode.Append, FileAccess.Write, FileShare.ReadWrite, bufferSize))
                {
                }
            }

            return new FileStream(fileName, FileMode.Append, FileAccess.Write, FileShare.ReadWrite, bufferSize);
        }

        public bool CollectionExists(ulong collectionId)
        {
            return File.Exists(Path.Combine(Dir, collectionId + ".docs"));
        }

        public void Dispose()
        {
            foreach (var file in _mmfs.Values)
                file.Dispose();
        }
    }
}