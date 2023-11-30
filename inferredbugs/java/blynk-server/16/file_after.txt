package cc.blynk.server.application.handlers.main.logic;

import cc.blynk.server.Holder;
import cc.blynk.server.core.BlockingIOProcessor;
import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.server.core.model.device.Device;
import cc.blynk.server.core.protocol.exceptions.IllegalCommandBodyException;
import cc.blynk.server.core.protocol.model.messages.StringMessage;
import cc.blynk.server.notifications.mail.MailWrapper;
import cc.blynk.utils.ParseUtil;
import cc.blynk.utils.StringUtils;
import io.netty.channel.Channel;
import io.netty.channel.ChannelHandlerContext;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import static cc.blynk.server.core.protocol.enums.Response.NOTIFICATION_ERROR;
import static cc.blynk.utils.BlynkByteBufUtil.makeResponse;
import static cc.blynk.utils.BlynkByteBufUtil.ok;

/**
 * Sends email from application.
 *
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 2/1/2015.
 *
 */
public class AppMailLogic {

    private static final Logger log = LogManager.getLogger(AppMailLogic.class);
    private final String BODY;

    private final BlockingIOProcessor blockingIOProcessor;
    private final MailWrapper mailWrapper;

    public AppMailLogic(Holder holder) {
        this.blockingIOProcessor = holder.blockingIOProcessor;
        this.BODY = blockingIOProcessor.tokenBody;
        this.mailWrapper =  holder.mailWrapper;
    }

    public void messageReceived(ChannelHandlerContext ctx, User user, StringMessage message) {
        String[] split = StringUtils.split2(message.body);

        int dashId = ParseUtil.parseInt(split[0]);
        DashBoard dash = user.profile.getDashByIdOrThrow(dashId);

        if (split.length == 2) {
            int deviceId = ParseUtil.parseInt(split[1]);
            Device device = dash.getDeviceById(deviceId);

            if (device == null || device.token == null) {
                throw new IllegalCommandBodyException("Wrong device id.");
            }

            makeSingleTokenEmail(ctx, dash, device, user.name, message.id);
        } else {
            if (dash.devices.length == 1) {
                makeSingleTokenEmail(ctx, dash, dash.devices[0], user.name, message.id);
            } else {
                sendMultiTokenEmail(ctx, dash, user.name, message.id);
            }

        }
    }

    private void makeSingleTokenEmail(ChannelHandlerContext ctx, DashBoard dash, Device device, String to, int msgId) {
        String dashName = dash.name == null ? "New Project" : dash.name;
        String deviceName = device.name == null ? "New Device" : device.name;
        String subj = "Auth Token for " + dashName + " project and device " + deviceName;
        String body = "Auth Token : " + device.token;

        log.trace("Sending single token mail for user {}, with token : '{}'.", to, device.token);
        mail(ctx.channel(), to, to, subj, body + BODY, msgId);
    }

    private void sendMultiTokenEmail(ChannelHandlerContext ctx, DashBoard dash, String to, int msgId) {
        String dashName = dash.name == null ? "New Project" : dash.name;
        String subj = "Auth Tokens for " + dashName + " project and " + dash.devices.length + " devices";

        StringBuilder body = new StringBuilder();
        for (Device device : dash.devices) {
            body.append("Auth Token for device '")
                .append(device.name == null ? "New Device" : device.name)
                .append("' : ")
                .append(device.token)
                .append("\n");
        }

        body.append(BODY);

        log.trace("Sending multi tokens mail for user {}, with {} tokens.", to, dash.devices.length);
        mail(ctx.channel(), to, to, subj, body.toString(), msgId);
    }

    private void mail(Channel channel, String username, String to, String subj, String body, int msgId) {
        blockingIOProcessor.execute(() -> {
            try {
                mailWrapper.sendText(to, subj, body);
                channel.writeAndFlush(ok(msgId), channel.voidPromise());
            } catch (Exception e) {
                log.error("Error sending email from application. From user {}, to : {}. Reason : {}",  username, to, e.getMessage());
                channel.writeAndFlush(makeResponse(msgId, NOTIFICATION_ERROR), channel.voidPromise());
            }
        });
    }
}
