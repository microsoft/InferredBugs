using System.IO;
using System.Linq;
using NUnit.Framework;
using Resin;

namespace Tests
{
    [TestFixture]
    public class FieldReaderTests
    {
        [Test]
        public void Can_read_field_file()
        {
            const string fileName = "c:\\temp\\resin_tests\\Can_read_field_file\\0.fld";
            if (File.Exists(fileName)) File.Delete(fileName);
            using (var fw = new FieldFile(fileName))
            {
                fw.Write(0, "hello", 0);
                fw.Write(5, "world", 1);
            }
            var reader = FieldReader.Load(fileName);
            var helloPositionsForDocId0 = reader.GetDocPosition("hello")[0];
            var worldPositionsForDocId5 = reader.GetDocPosition("world")[5];
            Assert.AreEqual(0, helloPositionsForDocId0.First());
            Assert.AreEqual(1, worldPositionsForDocId5.First());
        }
    }
}
