    public void testCpioUnarchive() throws Exception {
        final File output = new File(dir, "bla.cpio");
        {
            final File file1 = getFile("test1.xml");
            final File file2 = getFile("test2.xml");

            final OutputStream out = new FileOutputStream(output);
            final ArchiveOutputStream os = new ArchiveStreamFactory().createArchiveOutputStream("cpio", out);
            os.putArchiveEntry(new CpioArchiveEntry("test1.xml", file1.length()));
            IOUtils.copy(new FileInputStream(file1), os);
            os.closeArchiveEntry();

            os.putArchiveEntry(new CpioArchiveEntry("test2.xml", file2.length()));
            IOUtils.copy(new FileInputStream(file2), os);
            os.closeArchiveEntry();

            os.close();
            out.close();
        }

        // Unarchive Operation
        final File input = output;
        final InputStream is = new FileInputStream(input);
        final ArchiveInputStream in = new ArchiveStreamFactory().createArchiveInputStream("cpio", is);


        Map result = new HashMap();
        ArchiveEntry entry = null;
        while ((entry = in.getNextEntry()) != null) {
            File target = new File(dir, entry.getName());
            final OutputStream out = new FileOutputStream(target);
            IOUtils.copy(in, out);
            out.close();
            result.put(entry.getName(), target);
        }
        in.close();

        int lineSepLength = System.getProperty("line.separator").length();

        File t = (File)result.get("test1.xml");
        assertTrue("Expected " + t.getAbsolutePath() + " to exist", t.exists());
        assertEquals("length of " + t.getAbsolutePath(),
                     72 + 4 * lineSepLength, t.length());

        t = (File)result.get("test2.xml");
        assertTrue("Expected " + t.getAbsolutePath() + " to exist", t.exists());
        assertEquals("length of " + t.getAbsolutePath(),
                     73 + 5 * lineSepLength, t.length());
    }