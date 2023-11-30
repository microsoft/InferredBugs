package cc.blynk.server.notifications.push;

import cc.blynk.server.notifications.push.android.AndroidGCMMessage;
import cc.blynk.server.notifications.push.enums.Priority;
import cc.blynk.server.notifications.push.ios.IOSGCMMessage;
import com.fasterxml.jackson.core.JsonProcessingException;
import org.junit.Ignore;
import org.junit.Test;

import static org.junit.Assert.assertEquals;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 26.06.15.
 */
public class GCMWrapperTest {

    @Test
    @Ignore
    public void testIOS() throws Exception {
        GCMWrapper gcmWrapper = new GCMWrapper(null, null);
        gcmWrapper.send(new IOSGCMMessage("to", Priority.normal, "yo!!!", 1), null, null);
    }

    @Test
    @Ignore
    public void testAndroid() throws Exception {
        GCMWrapper gcmWrapper = new GCMWrapper(null, null);
        gcmWrapper.send(new AndroidGCMMessage("", Priority.normal, "yo!!!", 1), null, null);
    }

    @Test
    public void testValidAndroidJson() throws JsonProcessingException {
        assertEquals("{\"to\":\"to\",\"priority\":\"normal\",\"data\":{\"message\":\"yo!!!\",\"dashId\":1}}", new AndroidGCMMessage("to", Priority.normal, "yo!!!", 1).toJson());
    }

    @Test
    public void testValidIOSJson() throws JsonProcessingException {
        assertEquals("{\"to\":\"to\",\"priority\":\"normal\",\"notification\":{\"title\":\"Blynk Notification\",\"body\":\"yo!!!\",\"dashId\":1,\"sound\":\"default\"}}", new IOSGCMMessage("to", Priority.normal, "yo!!!", 1).toJson());
    }

}
