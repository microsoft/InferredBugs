package cc.blynk.server;

import java.net.URL;
import java.security.CodeSource;
import java.util.ArrayList;
import java.util.List;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

/**
 * Returns list of resources that were packed to jar
 *
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 11.12.15.
 */
public class JarWalker {

    public static List<String> find(String staticResourcesFolder) throws Exception {
        CodeSource src = ServerLauncher.class.getProtectionDomain().getCodeSource();
        List<String> staticResources = new ArrayList<>();

        if (src != null) {
            URL jar = src.getLocation();
            try (ZipInputStream zip = new ZipInputStream(jar.openStream())) {
                ZipEntry ze;

                while ((ze = zip.getNextEntry()) != null) {
                    String entryName = ze.getName();
                    if (entryName.startsWith(staticResourcesFolder) &&
                            (entryName.endsWith(".js") ||
                                    entryName.endsWith(".css") ||
                                    entryName.endsWith(".html")) ||
                            entryName.endsWith(".ico") ||
                            entryName.endsWith(".png")) {
                        staticResources.add(entryName);
                    }
                }
            }
        }

        return staticResources;
    }

}
