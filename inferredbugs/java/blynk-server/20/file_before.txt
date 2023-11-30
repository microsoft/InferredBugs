package cc.blynk.integration.tools;

import cc.blynk.server.core.BlockingIOProcessor;
import cc.blynk.server.core.model.AppName;
import cc.blynk.server.db.DBManager;
import cc.blynk.server.db.model.FlashedToken;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 01.03.17.
 */
public class FlahsedTokenGenerator {

    public static void main(String[] args) {
        List<FlashedToken> flashedTokens = generateTokens(10, AppName.BLYNK, 1);
        DBManager dbManager = new DBManager("db-test.properties", new BlockingIOProcessor(1, 100, null));
        dbManager.insertFlashedTokens(flashedTokens);
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

}
