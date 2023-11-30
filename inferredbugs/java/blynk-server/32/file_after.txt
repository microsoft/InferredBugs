package cc.blynk.integration.tools;

import cc.blynk.server.core.dao.TokenManager;
import cc.blynk.server.core.model.AppName;
import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.server.core.model.enums.Theme;
import cc.blynk.utils.JsonParser;
import cc.blynk.utils.SHA256Util;

import java.io.BufferedWriter;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 01.11.16.
 */
public class ProjectTokenGenerator {

    public static void main(String[] args) throws Exception {
        TokenManager tokenManager = new TokenManager(new ConcurrentHashMap<>(), null, null, "");
        String email = "dmitriy@blynk.cc";
        String pass = "b";
        String appName = AppName.BLYNK;
        User user = new User(email, SHA256Util.makeHash(pass, email), appName, "local", false, false);
        user.purchaseEnergy(98000);

        int count = 300;

        user.profile.dashBoards = new DashBoard[count];
        for (int i = 1; i <= count; i++) {
            DashBoard dash = new DashBoard();
            dash.id = i;
            dash.theme = Theme.Blynk;
            dash.isActive = true;
            user.profile.dashBoards[i - 1] = dash;
        }

        List<String> tokens = new ArrayList<>();
        for (int i = 1; i <= count; i++) {
            //tokens.add(tokenManager.refreshToken(user, i, 0));
        }

        write("/path/300_tokens.txt", tokens);

        write(Paths.get("/path/" + email + "." + appName + ".user"), JsonParser.toJson(user));

        //scp /path/dmitriy@blynk.cc.Blynk.user root@IP:/root/data/

    }

    private static void write(String outputPath, Collection<String> tokens) throws IOException {
        Path path = Paths.get(outputPath);
        write(path, tokens);
    }

    private static void write(Path path, Collection<String> tokens) throws IOException {
        try (BufferedWriter writer = Files.newBufferedWriter(path)) {
            for (String token : tokens) {
                writer.write(token);
                writer.newLine();
            }
        }
    }

    private static void write(Path path, String profile) throws IOException {
        try (BufferedWriter writer = Files.newBufferedWriter(path)) {
            writer.write(profile);
        }
    }

}
