package io.everitoken.sdk.java.abi;

import io.everitoken.sdk.java.apiResource.AbiBin;
import io.everitoken.sdk.java.exceptions.AbiSerialisationFailureException;
import io.everitoken.sdk.java.exceptions.ApiResponseException;
import io.everitoken.sdk.java.param.NetParams;
import io.everitoken.sdk.java.param.RequestParams;
import org.json.JSONException;
import org.json.JSONObject;

public class RemoteAbiSerialisationProvider implements AbiSerialisationProviderInterface {
    private final NetParams netParams;

    public RemoteAbiSerialisationProvider(NetParams netParams) {
        this.netParams = netParams;
    }

    @Override
    public String serialize(String data) {
        try {
            AbiBin abiBin = new AbiBin();
            JSONObject res = abiBin.request(RequestParams.of(netParams, () -> data));
            return res.getString("binargs");
        } catch (JSONException ex) {
            throw new IllegalArgumentException(String.format("Invalid json \"%s\" passed in.", data), ex);
        } catch (ApiResponseException ex) {
            throw new AbiSerialisationFailureException(ex.getRaw().toString(), ex);
        }
    }
}
