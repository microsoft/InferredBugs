package cc.blynk.server.admin.http.logic;

import cc.blynk.core.http.AuthHeadersBaseHttpHandler;
import cc.blynk.core.http.Response;
import cc.blynk.core.http.annotation.*;
import cc.blynk.server.Holder;
import cc.blynk.server.core.dao.TokenValue;
import cc.blynk.server.core.dao.UserKey;
import cc.blynk.server.core.dao.ota.OTAManager;
import cc.blynk.server.core.model.auth.Session;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.server.core.model.device.Device;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;

import static cc.blynk.core.http.Response.badRequest;
import static cc.blynk.core.http.Response.ok;
import static cc.blynk.server.core.protocol.enums.Command.*;


/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 03.12.15.
 */
@Path("/ota")
@ChannelHandler.Sharable
public class OTALogic extends AuthHeadersBaseHttpHandler {

    private final OTAManager otaManager;

    private static final String OTA_DIR = "/static/ota/";

    public OTALogic(Holder holder, String rootPath) {
        super(holder, rootPath);
        this.otaManager = holder.otaManager;
    }

    @GET
    @Path("/start")
    @Metric(HTTP_STOP_OTA)
    public Response startOTA(@QueryParam("fileName") String filename,
                             @QueryParam("token") String token) {
        TokenValue tokenValue = tokenManager.getTokenValueByToken(token);

        if (tokenValue == null) {
            log.debug("Requested token {} not found.", token);
            return badRequest("Invalid token.");
        }

        User user = tokenValue.user;
        int deviceId = tokenValue.deviceId;

        if (user == null) {
            return badRequest("Invalid auth credentials.");
        }

        Session session = sessionDao.userSession.get(new UserKey(user));
        if (session == null) {
            log.debug("No session for user {}.", user.email);
            return badRequest("Device wasn't connected yet.");
        }

        String otaFile = OTA_DIR + (filename == null ? "firmware_ota.bin" : filename);
        String body = otaManager.buildOTAInitCommandBody(otaFile);
        if (session.sendMessageToHardware(BLYNK_INTERNAL, 7777, body)) {
            log.debug("No device in session.");
            return badRequest("No device in session.");
        }

        Device device = tokenValue.dash.getDeviceById(deviceId);

        device.updateOTAInfo(user.email);

        return ok();
    }

    @GET
    @Path("/stop")
    @Metric(HTTP_START_OTA)
    public Response stopOTA(@Context ChannelHandlerContext ctx) {
        User initiator = ctx.channel().attr(AuthHeadersBaseHttpHandler.USER).get();
        otaManager.stop(initiator);

        return ok();
    }

}
