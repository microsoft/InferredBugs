package net.dongliu.requests.json;


import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import javax.annotation.Nonnull;
import javax.annotation.Nullable;
import javax.annotation.concurrent.ThreadSafe;
import java.util.Objects;

/**
 * Lookup json, from classpath
 *
 * @author Liu Dong
 */
@ThreadSafe
public class JsonLookup {
    private static final Logger logger = LogManager.getLogger();
    private static JsonLookup instance = new JsonLookup();
    @Nullable
    private volatile JsonProvider registeredJsonProvider;

    private JsonLookup() {
    }

    public static JsonLookup getInstance() {
        return instance;
    }

    /**
     * Set json provider for using.
     *
     * @see JsonProvider
     * @see JacksonProvider
     * @see GsonProvider
     */
    public void register(JsonProvider jsonProvider) {
        this.registeredJsonProvider = Objects.requireNonNull(jsonProvider);
        logger.debug(() -> "Set json provider to " + jsonProvider.getClass().getName());
    }

    /**
     * If classpath has gson
     */
    boolean hasGson() {
        try {
            Class.forName("com.google.gson.Gson");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }

    /**
     * Create Gson Provider
     */
    JsonProvider gsonProvider() {
        return new GsonProvider();
    }

    /**
     * if jackson in classpath
     */
    boolean hasJackson() {
        try {
            Class.forName("com.fasterxml.jackson.databind.ObjectMapper");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }

    boolean hasFastJson() {
        try {
            Class.forName("com.alibaba.fastjson.JSON");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }

    /**
     * Create Jackson Provider
     */
    JsonProvider jacksonProvider() {
        return new JacksonProvider();
    }

    JsonProvider fastJsonProvider() {
        return new FastJsonProvider();
    }

    /**
     * Find one json provider.
     *
     * @throws ProviderNotFoundException if no json provider found
     */
    @Nonnull
    public JsonProvider lookup() {
        if (registeredJsonProvider != null) {
            return registeredJsonProvider;
        }

        if (!init) {
            synchronized (this) {
                if (!init) {
                    lookedJsonProvider = lookupInClasspath();
                    init = true;
                }
            }
        }

        if (lookedJsonProvider != null) {
            return lookedJsonProvider;
        }
        throw new ProviderNotFoundException("Json Provider not found");
    }

    @Nullable
    private JsonProvider lookedJsonProvider;
    private boolean init;

    @Nullable
    private JsonProvider lookupInClasspath() {
        if (hasJackson()) {
            logger.debug("Use default jackson provider to deal with json");
            return jacksonProvider();
        }
        if (hasGson()) {
            logger.debug("Use default gson provider to deal with json");
            return gsonProvider();
        }
        if (hasFastJson()) {
            logger.debug("Use default fastJson provider to deal with json");
            return fastJsonProvider();
        }
        return null;
    }
}
