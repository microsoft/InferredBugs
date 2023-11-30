package cc.blynk.server.api.http.logic;

import cc.blynk.core.http.MediaType;
import cc.blynk.core.http.Response;
import cc.blynk.core.http.annotation.Consumes;
import cc.blynk.core.http.annotation.GET;
import cc.blynk.core.http.annotation.POST;
import cc.blynk.core.http.annotation.PUT;
import cc.blynk.core.http.annotation.Path;
import cc.blynk.core.http.annotation.PathParam;
import cc.blynk.core.http.annotation.QueryParam;
import cc.blynk.server.Holder;
import cc.blynk.server.api.http.logic.serialization.NotificationCloneHideFields;
import cc.blynk.server.api.http.logic.serialization.TwitterCloneHideFields;
import cc.blynk.server.api.http.pojo.EmailPojo;
import cc.blynk.server.api.http.pojo.PinData;
import cc.blynk.server.api.http.pojo.PushMessagePojo;
import cc.blynk.server.api.http.pojo.att.AttData;
import cc.blynk.server.api.http.pojo.att.AttValue;
import cc.blynk.server.core.BlockingIOProcessor;
import cc.blynk.server.core.dao.ReportingDao;
import cc.blynk.server.core.dao.SessionDao;
import cc.blynk.server.core.dao.TokenManager;
import cc.blynk.server.core.dao.UserKey;
import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.Pin;
import cc.blynk.server.core.model.auth.Session;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.server.core.model.enums.PinType;
import cc.blynk.server.core.model.widgets.MultiPinWidget;
import cc.blynk.server.core.model.widgets.OnePinWidget;
import cc.blynk.server.core.model.widgets.Widget;
import cc.blynk.server.core.model.widgets.notifications.Mail;
import cc.blynk.server.core.model.widgets.notifications.Notification;
import cc.blynk.server.core.model.widgets.notifications.Twitter;
import cc.blynk.server.core.processors.EventorProcessor;
import cc.blynk.server.core.protocol.exceptions.IllegalCommandBodyException;
import cc.blynk.server.core.protocol.exceptions.NoDataException;
import cc.blynk.server.core.stats.GlobalStats;
import cc.blynk.server.notifications.mail.MailWrapper;
import cc.blynk.server.notifications.push.GCMWrapper;
import cc.blynk.utils.ByteUtils;
import cc.blynk.utils.FileUtils;
import cc.blynk.utils.JsonParser;
import cc.blynk.utils.ReflectionUtil;
import cc.blynk.utils.StringUtils;
import com.fasterxml.jackson.databind.ObjectWriter;
import net.glxn.qrgen.core.image.ImageType;
import net.glxn.qrgen.javase.QRCode;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.ArrayList;
import java.util.Base64;
import java.util.List;
import java.util.Map;

import static cc.blynk.core.http.Response.ok;
import static cc.blynk.core.http.Response.redirect;
import static cc.blynk.server.core.protocol.enums.Command.HARDWARE;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_EMAIL;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_GET_DATA;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_GET_PIN_DATA;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_GET_PROJECT;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_IS_APP_CONNECTED;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_IS_HARDWARE_CONNECTED;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_NOTIFY;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_QR;
import static cc.blynk.server.core.protocol.enums.Command.HTTP_UPDATE_PIN_DATA;
import static cc.blynk.server.core.protocol.enums.Command.SET_WIDGET_PROPERTY;
import static cc.blynk.utils.StringUtils.BODY_SEPARATOR_STRING;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 25.12.15.
 */
@Path("/")
public class HttpAPILogic {

    protected static final ObjectWriter dashboardCloneWriter = JsonParser.init()
            .addMixIn(Twitter.class, TwitterCloneHideFields.class)
            .addMixIn(Notification.class, NotificationCloneHideFields.class)
            .writerFor(DashBoard.class);
    private static final Logger log = LogManager.getLogger(HttpAPILogic.class);
    private final TokenManager tokenManager;
    private final BlockingIOProcessor blockingIOProcessor;
    private final SessionDao sessionDao;
    private final GlobalStats globalStats;
    private final MailWrapper mailWrapper;
    private final GCMWrapper gcmWrapper;
    private final ReportingDao reportingDao;
    private final EventorProcessor eventorProcessor;

    public HttpAPILogic(Holder holder) {
        this(holder.tokenManager, holder.sessionDao, holder.blockingIOProcessor,
                holder.mailWrapper, holder.gcmWrapper, holder.reportingDao,
                holder.stats, holder.eventorProcessor);
    }

    private HttpAPILogic(TokenManager tokenManager, SessionDao sessionDao, BlockingIOProcessor blockingIOProcessor,
                         MailWrapper mailWrapper, GCMWrapper gcmWrapper, ReportingDao reportingDao,
                         GlobalStats globalStats, EventorProcessor eventorProcessor) {
        this.tokenManager = tokenManager;
        this.blockingIOProcessor = blockingIOProcessor;
        this.sessionDao = sessionDao;
        this.globalStats = globalStats;
        this.mailWrapper = mailWrapper;
        this.gcmWrapper = gcmWrapper;
        this.reportingDao = reportingDao;
        this.eventorProcessor = eventorProcessor;
    }

    private static String makeBody(DashBoard dash, byte pin, PinType pinType, String pinValue) {
        Widget widget = dash.findWidgetByPin(pin, pinType);
        if (widget == null) {
            return Pin.makeHardwareBody(pinType, pin, pinValue);
        } else {
            if (widget instanceof OnePinWidget) {
                return ((OnePinWidget) widget).makeHardwareBody();
            } else {
                return ((MultiPinWidget) widget).makeHardwareBody(pin, pinType);
            }
        }
    }

    @GET
    @Path("{token}/project")
    public Response getDashboard(@PathParam("token") String token) {
        globalStats.mark(HTTP_GET_PROJECT);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.error("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.error("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        DashBoard dashBoard = user.profile.getDashById(dashId);

        return ok(dashBoard.toString());
    }

    @GET
    @Path("{token}/isHardwareConnected")
    public Response isHardwareConnected(@PathParam("token") String token) {
        globalStats.mark(HTTP_IS_HARDWARE_CONNECTED);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.error("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.error("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        final Session session = sessionDao.userSession.get(new UserKey(user));

        return ok(session.isHardwareConnected(dashId));
    }

    @GET
    @Path("{token}/isAppConnected")
    public Response isAppConnected(@PathParam("token") String token) {
        globalStats.mark(HTTP_IS_APP_CONNECTED);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        final DashBoard dashBoard = user.profile.getDashById(dashId);

        final Session session = sessionDao.userSession.get(new UserKey(user));

        return ok(dashBoard.isActive && session.isAppConnected());
    }

    @GET
    @Path("{token}/get/{pin}")
    public Response getWidgetPinDataNew(@PathParam("token") String token,
                                     @PathParam("pin") String pinString) {
        return getWidgetPinData(token, pinString);
    }

    //todo old API.
    @GET
    @Path("{token}/pin/{pin}")
    public Response getWidgetPinData(@PathParam("token") String token,
                                     @PathParam("pin") String pinString) {

        globalStats.mark(HTTP_GET_PIN_DATA);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        DashBoard dashBoard = user.profile.getDashById(dashId);

        PinType pinType;
        byte pin;

        try {
            pinType = PinType.getPinType(pinString.charAt(0));
            pin = Byte.parseByte(pinString.substring(1));
        } catch (NumberFormatException | IllegalCommandBodyException e) {
            log.debug("Wrong pin format. {}", pinString);
            return Response.badRequest("Wrong pin format.");
        }

        Widget widget = dashBoard.findWidgetByPin(pin, pinType);

        if (widget == null) {
            String value = dashBoard.storagePins.get("" + pinType.pintTypeChar + pin);
            if (value == null) {
                log.debug("Requested pin {} not found. User {}", pinString, user.name);
                return Response.badRequest("Requested pin not exists in app.");
            }
            return ok(JsonParser.valueToJsonAsString(value.split(StringUtils.BODY_SEPARATOR_STRING)));
        }

        return ok(widget.getJsonValue());
    }

    @GET
    @Path("{token}/qr")
    //todo cover with test
    public Response getQR(@PathParam("token") String token) {
        globalStats.mark(HTTP_QR);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        DashBoard dashBoard = user.profile.getDashById(dashId);

        try {
            byte[] compressed = ByteUtils.compress(dashboardCloneWriter.writeValueAsString(dashBoard));
            String qrData = "bp1" + Base64.getEncoder().encodeToString(compressed);
            byte[] qrDataBinary = QRCode.from(qrData).to(ImageType.PNG).withSize(500, 500).stream().toByteArray();
            return ok(qrDataBinary, "image/png");
        } catch (Throwable e) {
            log.error("Error generating QR. Reason : {}", e.getMessage());
            return Response.badRequest("Error generating QR.");
        }
    }

    @GET
    @Path("{token}/data/{pin}")
    public Response getPinHistoryData(@PathParam("token") String token,
                                      @PathParam("pin") String pinString) {
        globalStats.mark(HTTP_GET_DATA);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        PinType pinType;
        byte pin;

        try {
            pinType = PinType.getPinType(pinString.charAt(0));
            pin = Byte.parseByte(pinString.substring(1));
        } catch (NumberFormatException | IllegalCommandBodyException e) {
            log.debug("Wrong pin format. {}", pinString);
            return Response.badRequest("Wrong pin format.");
        }

        //todo may be optimized
        try {
            java.nio.file.Path path = FileUtils.createCSV(reportingDao, user.name, dashId, pinType, pin);
            return redirect("/" + path.getFileName().toString());
        } catch (IllegalCommandBodyException e1) {
            log.debug(e1.getMessage());
            return Response.badRequest(e1.getMessage());
        } catch (NoDataException noData) {
            log.debug("No data for pin.");
            return Response.badRequest("No data for pin.");
        } catch (Exception e) {
            log.debug("Error getting pin data.");
            return Response.badRequest("Error getting pin data.");
        }
    }

    @POST
    @Path("{token}/custom/pin/{pin}")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response acceptCustomPostRequest(@PathParam("token") String token,
                                            @PathParam("pin") String pinString,
                                            AttData attData) {

        /**
         {
         "trigger":"forward everything to blynk",
         "timestamp":"2016-07-28T20:18:18.132Z",
         "event":"fired",
         "device":
         {
         "id":"cfeafd941b7dbfadf3a7dbd7e91d4d07",
         "name":"Arduino",
         "serial":null
         },
         "conditions":
         {
         "test":{"changed":true}
         },
         "values":
         {
         "test":
         {
         "value":"41",
         "timestamp":"2016-07-28T20:18:18.132Z",
         "unit":"?C"
         }
         },
         "timeframe":0,
         "custom_data":null
         }
         */

        List<String> valueList = new ArrayList<>();
        for (Map.Entry<String, AttValue> entries : attData.values.entrySet()) {
            valueList.add(entries.getValue().value);
        }
        return updateWidgetPinData(token, pinString, valueList.toArray(new String[valueList.size()]));
    }

    public Response updateWidgetProperty(String token,
                                         String pinString,
                                         String property,
                                         String... values) {
        globalStats.mark(HTTP_UPDATE_PIN_DATA);

        if (values.length == 0) {
            log.debug("No properties for update provided.");
            return Response.badRequest("No properties for update provided.");
        }

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        DashBoard dash = user.profile.getDashById(dashId);

        //todo add test for this use case
        if (!dash.isActive) {
            return Response.badRequest("Project is not active.");
        }

        PinType pinType;
        byte pin;
        try {
            pinType = PinType.getPinType(pinString.charAt(0));
            pin = Byte.parseByte(pinString.substring(1));
        } catch (NumberFormatException | IllegalCommandBodyException e) {
            log.debug("Wrong pin format. {}", pinString);
            return Response.badRequest("Wrong pin format.");
        }

        //for now supporting only virtual pins
        Widget widget = dash.findWidgetByPin(pin, pinType);

        if (widget == null || pinType != PinType.VIRTUAL) {
            log.debug("No widget for SetWidgetProperty command.");
            return Response.badRequest("No widget for SetWidgetProperty command.");
        }

        boolean isChanged;
        try {
            //todo for now supporting only single property
            isChanged = ReflectionUtil.setProperty(widget, property, values[0]);
        } catch (Exception e) {
            log.debug("Error setting widget property. Reason : {}", e.getMessage());
            return Response.badRequest("Error setting widget property.");
        }

        if (isChanged) {
            Session session = sessionDao.userSession.get(new UserKey(user));
            session.sendToApps(SET_WIDGET_PROPERTY, 111, dash.id + BODY_SEPARATOR_STRING +
                    pin + BODY_SEPARATOR_STRING + property + BODY_SEPARATOR_STRING + values[0]);
            return Response.ok();
        }

        return Response.badRequest("Error setting widget property.");
    }

    //todo it is a bit ugly right now. could be simplified by passing map of query params.
    @GET
    @Path("{token}/update/{pin}")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response updateWidgetPinDataViaGet(@PathParam("token") String token,
                                              @PathParam("pin") String pinString,
                                              @QueryParam("value") String[] pinValues,
                                              @QueryParam("label") String labelValue,
                                              @QueryParam("labels") String labelsValue,
                                              @QueryParam("color") String colorValue,
                                              @QueryParam("onLabel") String onLabelValue,
                                              @QueryParam("offLabel") String offLabelValue,
                                              @QueryParam("isOnPlay") String isOnPlay) {

        if (pinValues != null) {
            return updateWidgetPinData(token, pinString, pinValues);
        }
        if (labelValue != null) {
            return updateWidgetProperty(token, pinString, "label", labelValue);
        }
        if (labelsValue != null) {
            return updateWidgetProperty(token, pinString, "labels", labelsValue);
        }
        if (colorValue != null) {
            return updateWidgetProperty(token, pinString, "color", colorValue);
        }
        if (onLabelValue != null) {
            return updateWidgetProperty(token, pinString, "onLabel", onLabelValue);
        }
        if (offLabelValue != null) {
            return updateWidgetProperty(token, pinString, "offLabel", offLabelValue);
        }
        if (isOnPlay != null) {
            return updateWidgetProperty(token, pinString, "isOnPlay", isOnPlay);
        }

        return Response.badRequest("Wrong request format.");
    }

    @PUT
    @Path("{token}/update/{pin}")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response updateWidgetPinDataNew(@PathParam("token") String token,
                                        @PathParam("pin") String pinString,
                                        String[] pinValues) {
        return updateWidgetPinData(token, pinString, pinValues);
    }

    //todo remove later?
    @PUT
    @Path("{token}/pin/{pin}")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response updateWidgetPinData(@PathParam("token") String token,
                                        @PathParam("pin") String pinString,
                                        String[] pinValues) {

        globalStats.mark(HTTP_UPDATE_PIN_DATA);

        if (pinValues.length == 0) {
            log.debug("No pin for update provided.");
            return Response.badRequest("No pin for update provided.");
        }

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        DashBoard dash = user.profile.getDashById(dashId);

        PinType pinType;
        byte pin;

        try {
            pinType = PinType.getPinType(pinString.charAt(0));
            pin = Byte.parseByte(pinString.substring(1));
        } catch (NumberFormatException | IllegalCommandBodyException e) {
            log.debug("Wrong pin format. {}", pinString);
            return Response.badRequest("Wrong pin format.");
        }

        String pinValue = String.join(StringUtils.BODY_SEPARATOR_STRING, pinValues);

        reportingDao.process(user.name, dashId, pin, pinType, pinValue);

        dash.update(pin, pinType, pinValue);

        String body = makeBody(dash, pin, pinType, pinValue);

        Session session = sessionDao.userSession.get(new UserKey(user));
        if (session == null) {
            log.debug("No session for user {}.", user.name);
            return Response.ok();
        }

        eventorProcessor.process(session, dash, pin, pinType, pinValue);

        session.sendMessageToHardware(dashId, HARDWARE, 111, body);

        if (dash.isActive) {
            session.sendToApps(HARDWARE, 111, dashId + StringUtils.BODY_SEPARATOR_STRING + body);
        }

        return Response.ok();
    }

    @PUT
    @Path("{token}/extra/pin/{pin}")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response updateWidgetPinData(@PathParam("token") String token,
                                        @PathParam("pin") String pinString,
                                        PinData[] pinsData) {

        globalStats.mark(HTTP_UPDATE_PIN_DATA);

        if (pinsData.length == 0) {
            log.debug("No pin for update provided.");
            return Response.badRequest("No pin for update provided.");
        }

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        DashBoard dash = user.profile.getDashById(dashId);

        PinType pinType;
        byte pin;

        try {
            pinType = PinType.getPinType(pinString.charAt(0));
            pin = Byte.parseByte(pinString.substring(1));
        } catch (NumberFormatException | IllegalCommandBodyException e) {
            log.debug("Wrong pin format. {}", pinString);
            return Response.badRequest("Wrong pin format.");
        }

        for (PinData pinData : pinsData) {
            reportingDao.process(user.name, dashId, pin, pinType, pinData.value, pinData.timestamp);
        }

        dash.update(pin, pinType, pinsData[0].value);

        String body = makeBody(dash, pin, pinType, pinsData[0].value);

        if (body != null) {
            Session session = sessionDao.userSession.get(new UserKey(user));
            if (session == null) {
                log.error("No session for user {}.", user.name);
                return Response.ok();
            }
            session.sendMessageToHardware(dashId, HARDWARE, 111, body);

            if (dash.isActive) {
                session.sendToApps(HARDWARE, 111, dashId + StringUtils.BODY_SEPARATOR_STRING + body);
            }
        }

        return Response.ok();
    }

    @POST
    @Path("{token}/notify")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response notify(@PathParam("token") String token,
                                        PushMessagePojo message) {

        globalStats.mark(HTTP_NOTIFY);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        if (message == null || Notification.isWrongBody(message.body)) {
            log.debug("Notification body is wrong. '{}'", message == null ? "" : message.body);
            return Response.badRequest("Body is empty or larger than 255 chars.");
        }

        DashBoard dash = user.profile.getDashById(dashId);

        if (!dash.isActive) {
            log.debug("Project is not active.");
            return Response.badRequest("Project is not active.");
        }

        Notification notification = dash.getWidgetByType(Notification.class);

        if (notification == null || notification.hasNoToken()) {
            log.debug("No notification tokens.");
            return Response.badRequest("No notification widget or widget not initialized.");
        }

        log.trace("Sending push for user {}, with message : '{}'.", user.name, message.body);
        notification.push(gcmWrapper, message.body, 1);

        return Response.ok();
    }

    @POST
    @Path("{token}/email")
    @Consumes(value = MediaType.APPLICATION_JSON)
    public Response email(@PathParam("token") String token,
                                        EmailPojo message) {

        globalStats.mark(HTTP_EMAIL);

        User user = tokenManager.getUserByToken(token);

        if (user == null) {
            log.debug("Requested token {} not found.", token);
            return Response.badRequest("Invalid token.");
        }

        Integer dashId = user.getDashIdByToken(token);

        if (dashId == null) {
            log.debug("Dash id for token {} not found. User {}", token, user.name);
            return Response.badRequest("Didn't find dash id for token.");
        }

        DashBoard dash = user.profile.getDashById(dashId);

        if (!dash.isActive) {
            log.debug("Project is not active.");
            return Response.badRequest("Project is not active.");
        }

        Mail mail = dash.getWidgetByType(Mail.class);

        if (mail == null) {
            log.debug("No email widget.");
            return Response.badRequest("No email widget.");
        }

        if (message == null ||
                message.subj == null || message.subj.equals("") ||
                message.to == null || message.to.equals("")) {
            log.debug("Email body empty. '{}'", message);
            return Response.badRequest("Email body is wrong. Missing or empty fields 'to', 'subj'.");
        }

        log.trace("Sending Mail for user {}, with message : '{}'.", user.name, message.subj);
        mail(user.name, message.to, message.subj, message.title);

        return Response.ok();
    }

    private void mail(String username, String to, String subj, String body) {
        blockingIOProcessor.execute(() -> {
            try {
                mailWrapper.sendText(to, subj, body);
            } catch (Exception e) {
                log.error("Error sending email from HTTP. From : '{}', to : '{}'. Reason : {}", username, to, e.getMessage());
            }
        });
    }

}
