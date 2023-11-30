package cc.blynk.server.handlers.app;

import cc.blynk.common.model.messages.protocol.HardwareMessage;
import cc.blynk.server.dao.SessionDao;
import cc.blynk.server.dao.UserDao;
import cc.blynk.server.handlers.http.helpers.Response;
import cc.blynk.server.handlers.http.helpers.pojo.EmailPojo;
import cc.blynk.server.handlers.http.helpers.pojo.PushMessagePojo;
import cc.blynk.server.model.DashBoard;
import cc.blynk.server.model.HardwareBody;
import cc.blynk.server.model.auth.Session;
import cc.blynk.server.model.auth.User;
import cc.blynk.server.model.enums.PinType;
import cc.blynk.server.model.widgets.Widget;
import cc.blynk.server.model.widgets.notifications.Mail;
import cc.blynk.server.model.widgets.notifications.Notification;
import cc.blynk.server.workers.notifications.BlockingIOProcessor;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import javax.ws.rs.*;
import javax.ws.rs.core.MediaType;

import static cc.blynk.server.handlers.http.helpers.ResponseGenerator.*;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 25.12.15.
 */
@Path("/app")
public class AppHttpHandler {

    private static final Logger log = LogManager.getLogger(AppHttpHandler.class);

    private final UserDao userDao;
    private final BlockingIOProcessor blockingIOProcessor;
    private final SessionDao sessionDao;

    public AppHttpHandler(UserDao userDao, SessionDao sessionDao, BlockingIOProcessor blockingIOProcessor) {
        this.userDao = userDao;
        this.blockingIOProcessor = blockingIOProcessor;
        this.sessionDao = sessionDao;
    }

    @GET
    @Path("/{token}/widget/{pin}")
    public Response getWidgetPinData(@PathParam("token") String token,
                                     @PathParam("pin") String pinString) {

        User user = userDao.tokenManager.getUserByToken(token);

        if (user == null) {
            log.error("Requested token {} not found.", token);
            return Response.notFound();
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.error("Dash id for token {} not found. User {}", token, user.name);
            return Response.notFound();
        }

        DashBoard dashBoard = user.profile.getDashById(dashId);

        PinType pinType;
        byte pin;

        try {
            pinType = PinType.getPingType(pinString.charAt(0));
            pin = Byte.parseByte(pinString.substring(1));
        } catch (NumberFormatException e) {
            log.error("Wrong pin format. {}", pinString);
            return Response.notFound();
        }

        Widget widget = dashBoard.findWidgetByPin(pin, pinType);

        if (widget == null) {
            log.error("Requested pin {} not found. User {}", pinString, user.name);
            return Response.notFound();
        }

        return makeResponse(widget.getJsonValue());
    }

    @PUT
    @Path("/{token}/widget/{pin}")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response updateWidgetPinData(@PathParam("token") String token,
                                        @PathParam("pin") String pinString,
                                        String[] pinValues) {

        User user = userDao.tokenManager.getUserByToken(token);

        if (user == null) {
            log.error("Requested token {} not found.", token);
            return Response.notFound();
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.error("Dash id for token {} not found. User {}", token, user.name);
            return Response.notFound();
        }

        DashBoard dashBoard = user.profile.getDashById(dashId);

        PinType pinType;
        byte pin;

        try {
            pinType = PinType.getPingType(pinString.charAt(0));
            pin = Byte.parseByte(pinString.substring(1));
        } catch (NumberFormatException e) {
            log.error("Wrong pin format. {}", pinString);
            return Response.badRequest();
        }

        Widget widget = dashBoard.findWidgetByPin(pin, pinType);

        if (widget == null) {
            log.error("Requested pin {} not found. User {}", pinString, user.name);
            return Response.notFound();
        }

        widget.updateIfSame(new HardwareBody(pinType, pin, pinValues));

        String body = widget.makeHardwareBody();

        if (body != null) {
            Session session = sessionDao.getUserSession().get(user);
            session.sendMessageToHardware(dashId, new HardwareMessage(111, body));
        }

        return Response.noContent();
    }

    @POST
    @Path("/{token}/notify")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response updateWidgetPinData(@PathParam("token") String token,
                                        PushMessagePojo message) {

        User user = userDao.tokenManager.getUserByToken(token);

        if (user == null) {
            log.error("Requested token {} not found.", token);
            return Response.notFound();
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.error("Dash id for token {} not found. User {}", token, user.name);
            return Response.notFound();
        }

        if (message == null || Notification.isWrongBody(message.body)) {
            log.error("Notification body is wrong. '{}'", message == null ? "" : message.body);
            return Response.badRequest();
        }

        DashBoard dash = user.profile.getDashById(dashId);

        if (!dash.isActive) {
            log.error("No active dashboard.");
            return Response.notFound();
        }

        Notification notification = dash.getWidgetByType(Notification.class);

        if (notification == null || notification.hasNoToken()) {
            log.error("No notification tokens.");
            return Response.notFound();
        }

        log.trace("Sending push for user {}, with message : '{}'.", user.name, message.body);
        blockingIOProcessor.push(user, notification, message.body, 1);

        return Response.ok();
    }

    @POST
    @Path("/{token}/email")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response updateWidgetPinData(@PathParam("token") String token,
                                        EmailPojo message) {

        User user = userDao.tokenManager.getUserByToken(token);

        if (user == null) {
            log.error("Requested token {} not found.", token);
            return Response.notFound();
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.error("Dash id for token {} not found. User {}", token, user.name);
            return Response.notFound();
        }

        DashBoard dash = user.profile.getDashById(dashId);

        Mail mail = dash.getWidgetByType(Mail.class);

        if (mail == null || !dash.isActive) {
            log.error("No active dashboard.");
            return Response.notFound();
        }

        if (message == null ||
                message.subj == null || message.subj.equals("") ||
                message.to == null || message.to.equals("")) {
            log.error("Email body empty. '{}'", message);
            return Response.badRequest();
        }

        log.trace("Sending Mail for user {}, with message : '{}'.", user.name, message.subj);
        blockingIOProcessor.mail(user, message.to, message.subj, message.title);

        return Response.ok();
    }

}
