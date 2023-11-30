package cc.blynk.server.handlers.app.logic.sharing;

import cc.blynk.common.model.messages.Message;
import cc.blynk.server.dao.UserRegistry;
import cc.blynk.server.exceptions.InvalidTokenException;
import cc.blynk.server.model.DashBoard;
import cc.blynk.server.model.auth.User;
import cc.blynk.server.model.widgets.others.Notification;
import cc.blynk.server.model.widgets.others.Twitter;
import io.netty.channel.ChannelHandlerContext;

import java.util.Map;

import static cc.blynk.common.model.messages.MessageFactory.produce;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 2/1/2015.
 *
 */
public class GetSharedDashLogic {

    private final UserRegistry userRegistry;

    public GetSharedDashLogic(UserRegistry userRegistry) {
        this.userRegistry = userRegistry;
    }

    private static Integer getSharedDashId(Map<Integer, String> sharedTokens, String token) {
        for (Map.Entry<Integer, String> entry : sharedTokens.entrySet()) {
            if (entry.getValue().equals(token)) {
                return entry.getKey();
            }
        }

        return null;
    }

    private static void cleanPrivateData(DashBoard dashBoard) {
        Twitter twitter = dashBoard.getWidgetByType(Twitter.class);
        if (twitter != null) {
            twitter.cleanPrivateData();
        }

        Notification notification = dashBoard.getWidgetByType(Notification.class);
        if (notification != null) {
            notification.cleanPrivateData();
        }
    }

    public void messageReceived(ChannelHandlerContext ctx, Message message) {
        String token = message.body;

        User userThatShared = userRegistry.sharedTokenManager.getUserByToken(token);

        if (userThatShared == null) {
            throw new InvalidTokenException("Illegal sharing token. No user with those shared token.", message.id);
        }

        Integer dashId = getSharedDashId(userThatShared.dashShareTokens, token);

        if (dashId == null) {
            throw new InvalidTokenException("Illegal sharing token. User has not token. Could happen only in rare cases.", message.id);
        }

        DashBoard dashBoard = userThatShared.profile.getDashboardById(dashId);
        cleanPrivateData(dashBoard);

        ctx.writeAndFlush(produce(message.id, message.command, dashBoard.toString()));
    }

}