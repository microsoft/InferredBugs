using System.IO;
using System.Text;
using NUnit.Framework;
using Resin.IO;

namespace Tests
{
    [TestFixture]
    public class LcrsTrieTests
    {
        [Test]
        public void Can_scan_fuzzy_from_disk()
        {
            const string fileName = "1.bt";

            var tree = new LcrsTrie('\0', false);
            tree.Add("baby");
            tree.Add("bad");
            tree.Add("bank");
            tree.Add("box");
            tree.Add("dad");
            tree.Add("daddy");
            tree.Add("dance");
            tree.Add("dancing");

            tree.Serialize(fileName);

            using (var fs = File.OpenRead(fileName))
            using (var sr = new StreamReader(fs, Encoding.Unicode))
            using (var scanner = new LcrsTreeStreamReader(sr))
            {
                var near = scanner.Near("bazy", 1);
                Assert.AreEqual(1, near.Count);
                Assert.IsTrue(near.Contains("baby"));
            }

            using (var fs = File.OpenRead(fileName))
            using (var sr = new StreamReader(fs, Encoding.Unicode))
            using (var scanner = new LcrsTreeStreamReader(sr))
            {
                var near = scanner.Near("bazy", 2);
                Assert.AreEqual(3, near.Count);
                Assert.IsTrue(near.Contains("baby"));
                Assert.IsTrue(near.Contains("bank"));
                Assert.IsTrue(near.Contains("bad"));
            }
        }

        [Test]
        public void Can_scan_prefix_from_disk()
        {
            const string fileName = "1.bt";

            var tree = new LcrsTrie('\0', false);
            tree.Add("baby");
            tree.Add("bad");
            tree.Add("bank");
            tree.Add("box");
            tree.Add("dad");
            tree.Add("daddy");
            tree.Add("dance");
            tree.Add("dancing");

            tree.Serialize(fileName);

            using (var fs = File.OpenRead(fileName))
            using (var sr = new StreamReader(fs, Encoding.Unicode))
            using (var scanner = new LcrsTreeStreamReader(sr))
            {
                var startsWith = scanner.StartsWith("ba");
                Assert.AreEqual(3, startsWith.Count);
                Assert.IsTrue(startsWith.Contains("baby"));
                Assert.IsTrue(startsWith.Contains("bad"));
                Assert.IsTrue(startsWith.Contains("bank"));
            }

            using (var fs = File.OpenRead(fileName))
            using (var sr = new StreamReader(fs, Encoding.Unicode))
            using (var scanner = new LcrsTreeStreamReader(sr))
            {
                Assert.IsTrue(scanner.HasWord("baby"));
            }
        }

        [Test]
        public void Can_scan_exact_from_disk()
        {
            const string fileName = "0.bt";

            var tree = new LcrsTrie('\0', false);
            tree.Add("baby");
            tree.Add("bad");
            tree.Add("bank");
            tree.Add("box");
            tree.Add("dad");
            tree.Add("daddy");
            tree.Add("dance");
            tree.Add("dancing");

            tree.Serialize(fileName);

            using (var fs = File.OpenRead(fileName))
            using (var sr = new StreamReader(fs, Encoding.Unicode))
            using (var scanner = new LcrsTreeStreamReader(sr))
            {
                Assert.IsFalse(scanner.HasWord("bab"));
            }

            using (var fs = File.OpenRead(fileName))
            using (var sr = new StreamReader(fs, Encoding.Unicode))
            using (var scanner = new LcrsTreeStreamReader(sr))
            {
                Assert.IsTrue(scanner.HasWord("baby"));
            }
        }

        [Test]
        public void Can_serialize()
        {
            var tree = new LcrsTrie('\0', false);
            tree.Add("baby");
            tree.Add("bad");
            tree.Add("bank");
            tree.Add("box");
            tree.Add("dad");
            tree.Add("daddy");
            tree.Add("dance");
            tree.Add("dancing");

            tree.Serialize("0.bt");
            var acctual = File.ReadAllText("0.bt", Encoding.Unicode);

            const string expected = "d1100\r\na0101\r\nn1102\r\nc0103\r\ni1104\r\nn0105\r\ng0016\r\ne0014\r\nd0112\r\nd0103\r\ny0014\r\nb0100\r\no1101\r\nx0012\r\na0101\r\nn1102\r\nk0013\r\nd1012\r\nb0102\r\ny0013\r\n";

            Assert.AreEqual(expected, acctual);
        }

        [Test]
        public void Can_find_near()
        {
            var tree = new LcrsTrie('\0', false);
            var near = tree.Near("ba", 1);

            Assert.That(near, Is.Empty);

            tree.Add("bad");
            near = tree.Near("ba", 1);

            Assert.That(near.Count, Is.EqualTo(1));
            Assert.IsTrue(near.Contains("bad"));

            tree.Add("baby");
            near = tree.Near("ba", 1);

            Assert.That(near.Count, Is.EqualTo(1));
            Assert.IsTrue(near.Contains("bad"));

            tree.Add("b");
            near = tree.Near("ba", 1);

            Assert.That(near.Count, Is.EqualTo(2));
            Assert.IsTrue(near.Contains("bad"));
            Assert.IsTrue(near.Contains("b"));

            near = tree.Near("ba", 2);

            Assert.That(near.Count, Is.EqualTo(3));
            Assert.IsTrue(near.Contains("b"));
            Assert.IsTrue(near.Contains("bad"));
            Assert.IsTrue(near.Contains("baby"));

            near = tree.Near("ba", 0);

            Assert.That(near.Count, Is.EqualTo(0));

            tree.Add("bananas");
            near = tree.Near("ba", 6);

            Assert.That(near.Count, Is.EqualTo(4));
            Assert.IsTrue(near.Contains("b"));
            Assert.IsTrue(near.Contains("bad"));
            Assert.IsTrue(near.Contains("baby"));
            Assert.IsTrue(near.Contains("bananas"));

            near = tree.Near("bazy", 1);

            Assert.That(near.Count, Is.EqualTo(1));
            Assert.IsTrue(near.Contains("baby"));

            tree.Add("bank");
            near = tree.Near("bazy", 3);

            Assert.AreEqual(4, near.Count);
            Assert.IsTrue(near.Contains("baby"));
            Assert.IsTrue(near.Contains("bank"));
            Assert.IsTrue(near.Contains("bad"));
            Assert.IsTrue(near.Contains("b"));
        }

        [Test]
        public void Can_find_suffixes()
        {
            var tree = new LcrsTrie('\0', false);

            var prefixed = tree.StartsWith("ba");

            Assert.That(prefixed, Is.Empty);

            tree.Add("bad");

            prefixed = tree.StartsWith("ba");

            Assert.That(prefixed.Count, Is.EqualTo(1));
            Assert.IsTrue(prefixed.Contains("bad"));

            tree.Add("baby");

            prefixed = tree.StartsWith("ba");

            Assert.That(prefixed.Count, Is.EqualTo(2));
            Assert.IsTrue(prefixed.Contains("bad"));
            Assert.IsTrue(prefixed.Contains("baby"));
        }

        [Test]
        public void Can_find_exact()
        {
            var tree = new LcrsTrie('\0', false);

            Assert.False(tree.HasWord("xxx"));

            tree.Add("xxx");

            Assert.True(tree.HasWord("xxx"));
            Assert.False(tree.HasWord("baby"));
            Assert.False(tree.HasWord("dad"));

            tree.Add("baby");

            Assert.True(tree.HasWord("xxx"));
            Assert.True(tree.HasWord("baby"));
            Assert.False(tree.HasWord("dad"));

            tree.Add("dad");

            Assert.True(tree.HasWord("xxx"));
            Assert.True(tree.HasWord("baby"));
            Assert.True(tree.HasWord("dad"));
        }

        [Test]
        public void Can_build_one_leg()
        {
            var tree = new LcrsTrie('\0', false);
            
            tree.Add("baby");

            Assert.That(tree.LeftChild.Value, Is.EqualTo('b'));
            Assert.That(tree.LeftChild.LeftChild.Value, Is.EqualTo('a'));
            Assert.That(tree.LeftChild.LeftChild.LeftChild.Value, Is.EqualTo('b'));
            Assert.That(tree.LeftChild.LeftChild.LeftChild.LeftChild.Value, Is.EqualTo('y'));

            Assert.True(tree.HasWord("baby"));
        }

        [Test]
        public void Can_build_two_legs()
        {
            var root = new LcrsTrie('\0', false);

            root.Add("baby");
            root.Add("dad");

            Assert.That(root.LeftChild.Value, Is.EqualTo('d'));
            Assert.That(root.LeftChild.LeftChild.Value, Is.EqualTo('a'));
            Assert.That(root.LeftChild.LeftChild.LeftChild.Value, Is.EqualTo('d'));

            Assert.That(root.LeftChild.RightSibling.Value, Is.EqualTo('b'));
            Assert.That(root.LeftChild.RightSibling.LeftChild.Value, Is.EqualTo('a'));
            Assert.That(root.LeftChild.RightSibling.LeftChild.LeftChild.Value, Is.EqualTo('b'));
            Assert.That(root.LeftChild.RightSibling.LeftChild.LeftChild.LeftChild.Value, Is.EqualTo('y'));

            Assert.True(root.HasWord("baby"));
            Assert.True(root.HasWord("dad"));
        }

        [Test]
        public void Can_append()
        {
            var root = new LcrsTrie('\0', false);

            root.Add("baby");
            root.Add("bad");

            Assert.That(root.LeftChild.Value, Is.EqualTo('b'));
            Assert.That(root.LeftChild.LeftChild.Value, Is.EqualTo('a'));
            Assert.That(root.LeftChild.LeftChild.LeftChild.Value, Is.EqualTo('d'));

            Assert.That(root.LeftChild.LeftChild.LeftChild.RightSibling.Value, Is.EqualTo('b'));
            Assert.That(root.LeftChild.LeftChild.LeftChild.RightSibling.LeftChild.Value, Is.EqualTo('y'));

            Assert.True(root.HasWord("baby"));
            Assert.True(root.HasWord("bad"));
        }
    }
}