package cc.blynk.server.hardware.handlers.hardware.logic;

import cc.blynk.server.core.dao.SessionDao;
import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.auth.Session;
import cc.blynk.server.core.model.enums.PinType;
import cc.blynk.server.core.model.widgets.Widget;
import cc.blynk.server.core.protocol.exceptions.IllegalCommandException;
import cc.blynk.server.core.protocol.model.messages.StringMessage;
import cc.blynk.server.core.session.HardwareStateHolder;
import cc.blynk.utils.ParseUtil;
import cc.blynk.utils.ReflectionUtil;
import cc.blynk.utils.StringUtils;
import io.netty.channel.ChannelHandlerContext;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import static cc.blynk.server.core.protocol.enums.Command.*;
import static cc.blynk.server.core.protocol.enums.Response.*;
import static cc.blynk.utils.ByteBufUtil.*;

/**
 * Handler that allows to change widget properties from hardware side.
 *
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 2/1/2015.
 *
 */
public class SetWidgetPropertyLogic {

    private static final Logger log = LogManager.getLogger(SetWidgetPropertyLogic.class);

    private final SessionDao sessionDao;

    public SetWidgetPropertyLogic(SessionDao sessionDao) {
        this.sessionDao = sessionDao;
    }

    public void messageReceived(ChannelHandlerContext ctx, HardwareStateHolder state, StringMessage message) {
        Session session = sessionDao.userSession.get(state.user);

        String[] bodyParts = message.body.split(StringUtils.BODY_SEPARATOR_STRING);

        if (bodyParts.length != 3) {
            throw new IllegalCommandException("SetWidgetProperty command body has wrong format.");
        }

        byte pin = ParseUtil.parseByte(bodyParts[0]);
        String property = bodyParts[1];
        String propertyValue = bodyParts[2];

        if (property.length() == 0 || propertyValue.length() == 0) {
            throw new IllegalCommandException("SetWidgetProperty command body has wrong format.");
        }

        DashBoard dash = state.user.profile.getDashByIdOrThrow(state.dashId);

        //for now supporting only virtual pins
        Widget widget = dash.findWidgetByPin(pin, PinType.VIRTUAL);
        boolean isChanged;
        try {
            isChanged = ReflectionUtil.setProperty(widget, property, propertyValue);
        } catch (Exception e) {
            log.error("Error setting widget property. Reason : {}", e.getMessage());
            ctx.writeAndFlush(makeResponse(message.id, ILLEGAL_COMMAND_BODY), ctx.voidPromise());
            return;
        }

        if (isChanged) {
            if (dash.isActive) {
                session.sendToApps(SET_WIDGET_PROPERTY, message.id, dash.id + StringUtils.BODY_SEPARATOR_STRING + message.body);
            }

            ctx.writeAndFlush(ok(message.id), ctx.voidPromise());
        } else {
            ctx.writeAndFlush(makeResponse(message.id, ILLEGAL_COMMAND_BODY), ctx.voidPromise());
        }
    }

}
