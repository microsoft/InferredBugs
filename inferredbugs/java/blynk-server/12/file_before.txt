package cc.blynk.integration.http;

import cc.blynk.integration.BaseTest;
import cc.blynk.server.api.http.HttpAPIServer;
import cc.blynk.server.core.BaseServer;
import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.utils.JsonParser;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.junit.AfterClass;
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.runners.MockitoJUnitRunner;

import java.io.InputStream;

import static org.junit.Assert.assertEquals;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 24.12.15.
 */
@RunWith(MockitoJUnitRunner.class)
public class HttpAPIProjectTest extends BaseTest {

    private static BaseServer httpServer;
    private static CloseableHttpClient httpclient;
    private static String httpsServerUrl;
    private DashBoard dashBoard;

    @AfterClass
    public static void shutdown() throws Exception {
        httpclient.close();
        httpServer.close();
    }

    @Before
    public void init() throws Exception {
        if (httpServer == null) {
            httpServer = new HttpAPIServer(holder).start();
            httpsServerUrl = String.format("http://localhost:%s/", httpPort);
            httpclient = HttpClients.createDefault();
        }
    }

    @Override
    public String getDataFolder() {
        return getRelativeDataFolder("/profiles");
    }

    //----------------------------GET METHODS SECTION

    @Test
    public void testGetWithFakeToken() throws Exception {
        String token = "4ae3851817194e2596cf1b7103603ef8";
        HttpGet request = new HttpGet(httpsServerUrl + token + "/project");

        InputStream is = getClass().getResourceAsStream("/profiles/u_dmitriy@blynk.cc.user");
        User user = JsonParser.mapper.readValue(is, User.class);

        Integer dashId = user.getDashIdByToken(token);
        dashBoard = user.profile.getDashById(dashId);

        try (CloseableHttpResponse response = httpclient.execute(request)) {
            assertEquals(200, response.getStatusLine().getStatusCode());
            assertEquals(dashBoard.toString(), consumeText(response));
        }
    }


}
