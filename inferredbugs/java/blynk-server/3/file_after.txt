package cc.blynk.server.handlers;

import cc.blynk.common.model.messages.protocol.SaveProfileMessage;
import cc.blynk.server.auth.User;
import cc.blynk.server.auth.UserRegistry;
import cc.blynk.server.exceptions.InvalidCommandFormatException;
import cc.blynk.server.group.Session;
import cc.blynk.server.model.UserProfile;
import cc.blynk.server.utils.FileManager;
import cc.blynk.server.utils.JsonParser;
import io.netty.channel.ChannelHandlerContext;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import static cc.blynk.common.enums.Response.OK;
import static cc.blynk.common.enums.Response.SERVER_ERROR;
import static cc.blynk.common.model.messages.MessageFactory.produce;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 2/1/2015.
 *
 */
public class SaveProfileHandler extends BaseSimpleChannelInboundHandler<SaveProfileMessage> {

    private static final Logger log = LogManager.getLogger(SaveProfileHandler.class);

    public SaveProfileHandler(FileManager fileManager, UserRegistry userRegistry) {
        super(fileManager, userRegistry);
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, SaveProfileMessage message) throws Exception {
        String userProfileString = message.body;

        //expecting message with 2 parts
        if (userProfileString == null || userProfileString.equals("")) {
            throw new InvalidCommandFormatException("Save Profile Handler. Income profile message is empty.", message.id);
        }

        log.info("Trying to parseProfile user profile : {}", userProfileString);
        UserProfile userProfile = JsonParser.parseProfile(userProfileString);
        if (userProfile == null) {
            throw new InvalidCommandFormatException("Register Handler. Wrong user profile message format.", message.id);
        }

        log.info("Trying save user profile.");

        User authUser = Session.findUserByChannel(ctx.channel(), message.id);

        authUser.setUserProfile(userProfile);
        boolean profileSaved = fileManager.overrideUserFile(authUser);

        if (profileSaved) {
            ctx.writeAndFlush(produce(message.id, OK));
        } else {
            ctx.writeAndFlush(produce(message.id, SERVER_ERROR));
        }
    }


}
