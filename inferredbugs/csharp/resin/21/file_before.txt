using System.IO;

namespace DocumentTable
{
    public class ReadSessionFactory : IReadSessionFactory
    {
        private readonly string _directory;

        public ReadSessionFactory(string directory)
        {
            _directory = directory;
        }

        public IReadSession OpenReadSession(long version)
        {
            var ix = BatchInfo.Load(Path.Combine(_directory, version + ".ix"));
            var docFileName = Path.Combine(_directory, version + ".dtbl");
            var keyIndexFileName = Path.Combine(_directory, version + ".kix");
            var keyIndex = TableSerializer.GetKeyIndex(keyIndexFileName);
            var compoundFileName = Path.Combine(_directory, version + ".rdb");

            var compoundFile = new FileStream(
                compoundFileName, 
                FileMode.Open, 
                FileAccess.Read, 
                FileShare.Read, 
                4096 * 1, 
                FileOptions.RandomAccess);
            
            return new ReadSession(
                ix,
                new PostingsReader(compoundFile, ix.PostingsOffset),
                new DocHashReader(compoundFile, ix.DocHashOffset),
                new DocumentAddressReader(compoundFile, ix.DocAddressesOffset),
                new DocumentReader(
                    new FileStream(docFileName, FileMode.Open, FileAccess.Read, FileShare.Read, 4096 * 1, FileOptions.RandomAccess), 
                    ix.Compression, 
                    keyIndex),
                compoundFile);
        }
    }
}
