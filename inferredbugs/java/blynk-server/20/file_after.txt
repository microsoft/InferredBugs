package cc.blynk.integration.tools;

import cc.blynk.server.core.BlockingIOProcessor;
import cc.blynk.server.db.DBManager;
import cc.blynk.server.db.model.FlashedToken;
import net.glxn.qrgen.core.image.ImageType;
import net.glxn.qrgen.javase.QRCode;

import java.io.OutputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 01.03.17.
 */
public class FlahsedTokenGenerator {

    public static void main(String[] args) throws Exception{
        List<FlashedToken> flashedTokens = generateTokens(100, "Grow", 2);
        DBManager dbManager = new DBManager("db-test.properties", new BlockingIOProcessor(1, 100, null));

        dbManager.insertFlashedTokens(flashedTokens);

        for (FlashedToken token : flashedTokens) {
            Path path = Paths.get("/home/doom369/Downloads/grow",  token.token + "_" + token.deviceId + ".jpg");
            generateQR(token.token, path);
        }
    }

    private static List<FlashedToken> generateTokens(int count, String appName, int deviceCount) {
        List<FlashedToken> flashedTokens = new ArrayList<>();

        for (int deviceId = 0; deviceId < deviceCount; deviceId++) {
            for (int i = 0; i < count; i++) {
                String token = UUID.randomUUID().toString().replace("-", "");
                flashedTokens.add(new FlashedToken(token, appName, deviceId));
                System.out.println("Token : " + token + ", deviceId : " + deviceId + ", appName : " + appName);
            }
        }
        return flashedTokens;
    }

    private static void generateQR(String text, Path outputFile) throws Exception {
        try (OutputStream out = Files.newOutputStream(outputFile)) {
            QRCode.from(text).to(ImageType.JPG).writeTo(out);
        }
    }

}
