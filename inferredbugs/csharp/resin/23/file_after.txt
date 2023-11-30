using DocumentTable;
using Microsoft.VisualStudio.TestTools.UnitTesting;
using Resin;
using System;
using System.IO;

namespace Tests
{
    [TestClass()]
    public class Setup
    {
        private const string root = @"c:\temp\resin_tests\";

        protected static string CreateDir()
        {
            var dir = root + Guid.NewGuid().ToString();
            Directory.CreateDirectory(dir);
            return dir;
        }

        protected static IReadSession CreateReadSession(string directory, long version)
        {
            return new FullTextReadSessionFactory(directory).OpenReadSession(version);
        }

        [AssemblyInitialize()]
        public static void AssemblyInit(TestContext context)
        {
            if(Directory.Exists(root))
            foreach(var dir in Directory.GetDirectories(root))
            {
                Directory.Delete(dir, true);
            }
        }
    }
}