using Rhisis.Core.Structures;
using System;
using System.IO;

namespace Rhisis.Core.Resources
{
    /// <summary>
    /// FlyFF World File structure.
    /// </summary>
    public class WldFile : FileStream, IDisposable
    {
        public WldFileInformations WorldInformations { get; private set; }

        /// <summary>
        /// Creates a new <see cref="WldFile"/> instance.
        /// </summary>
        /// <param name="filePath">Wld file data</param>
        public WldFile(string filePath)
            : base(filePath, FileMode.Open, FileAccess.Read)
        {
            this.Read();
        }

        /// <summary>
        /// Reads the content of the Wld file.
        /// </summary>
        private void Read()
        {
            var reader = new StreamReader(this);
            Vector3 size = null;
            bool isIndoor = false;
            bool canFly = false;
            int mpu = 0;
            int revivalMapId = 0;
            string revivalKey = string.Empty;

            while (!reader.EndOfStream)
            {
                var line = reader.ReadLine().Trim().ToLower();

                if (line.StartsWith("//") || string.IsNullOrEmpty(line))
                    continue;

                string[] lineArray = line.Split(new char[] { ' ', '\t' }, StringSplitOptions.RemoveEmptyEntries);

                switch (lineArray[0].ToLower())
                {
                    case "size":
                        size = this.ReadSize(lineArray);
                        break;
                    case "indoor":
                        isIndoor = lineArray[1] == "1" ? true : false;
                        break;
                    case "fly":
                        canFly = lineArray[1] == "1" ? true : false;
                        break;
                    case "mpu":
                        mpu = int.Parse(lineArray[1]);
                        break;
                    case "revival":
                        revivalMapId = int.Parse(lineArray[1]);
                        revivalKey = lineArray[2].Trim('"');
                        break;
                }
            }

            this.WorldInformations = new WldFileInformations((int)size.X, (int)size.Z, mpu, isIndoor, canFly, revivalMapId, revivalKey);
        }

        /// <summary>
        /// Read the "size" field of the wld file.
        /// </summary>
        /// <param name="lineArray">Current line array</param>
        private Vector3 ReadSize(string[] lineArray)
        {
            string width = lineArray[1].Replace(",", string.Empty);
            string length = lineArray[2];

            return new Vector3(int.Parse(width), 0, int.Parse(length));
        }
    }
}
