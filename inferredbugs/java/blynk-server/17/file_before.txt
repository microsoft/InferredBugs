package cc.blynk.server.core.dao;

import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.server.core.model.device.Device;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 22.09.15.
 */
class RegularTokenManager {

    private static final Logger log = LogManager.getLogger(RegularTokenManager.class);

    private final ConcurrentMap<String, TokenValue> cache;

    public RegularTokenManager(Iterable<User> users) {
        this.cache = new ConcurrentHashMap<String, TokenValue>() {{
            for (User user : users) {
                if (user.profile != null) {
                    for (DashBoard dashBoard : user.profile.dashBoards) {
                        for (Device device : dashBoard.devices) {
                            put(device.token, new TokenValue(user, dashBoard.id, device.id));
                        }
                    }
                }
            }
        }};
    }

    String assignToken(User user, int dashId, int deviceId, String newToken) {
        // Clean old token from cache if exists.
        DashBoard dash = user.profile.getDashByIdOrThrow(dashId);
        Device device = dash.getDeviceById(deviceId);

        String oldToken = removeOldToken(device);

        //assign new token
        device.token = newToken;
        cache.put(newToken, new TokenValue(user, dashId, deviceId));

        user.lastModifiedTs = System.currentTimeMillis();

        log.debug("Generated token for user {}, dashId {}, deviceId {} is {}.", user.name, dashId, deviceId, newToken);

        return oldToken;
    }

    private String removeOldToken(Device device) {
        if (device != null && device.token != null) {
            cache.remove(device.token);
            return device.token;
        }
        return null;
    }

    public TokenValue getUserByToken(String token) {
        return cache.get(token);
    }

    String[] deleteProject(DashBoard dash) {
        String[] removedTokens = new String[dash.devices.length];
        for (int i = 0; i < dash.devices.length; i++) {
            Device device = dash.devices[i];
            if (device != null && device.token != null) {
                cache.remove(device.token);
                removedTokens[i] = device.token;
            }
        }
        return removedTokens;
    }

}
