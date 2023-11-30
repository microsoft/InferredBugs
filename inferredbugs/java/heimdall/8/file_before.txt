
package br.com.conductor.heimdall.gateway.service;

/*-
 * =========================LICENSE_START==================================
 * heimdall-gateway
 * ========================================================================
 * Copyright (C) 2018 Conductor Tecnologia SA
 * ========================================================================
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 * ==========================LICENSE_END===================================
 */

import br.com.conductor.heimdall.core.util.BeanManager;
import br.com.conductor.heimdall.gateway.filter.helper.HelperImpl;
import br.com.conductor.heimdall.middleware.spec.ApiResponse;
import br.com.conductor.heimdall.middleware.spec.Helper;
import com.netflix.zuul.context.RequestContext;
import org.assertj.core.util.Lists;
import org.redisson.api.RBucket;
import org.redisson.api.RedissonClient;
import org.springframework.stereotype.Service;

import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import static br.com.conductor.heimdall.core.util.ConstantsCache.CACHE_BUCKET;
import static br.com.conductor.heimdall.core.util.ConstantsCache.CACHE_TIME_TO_LIVE;

/**
 * Cache service provides methods to create and delete a response cache from a request.
 *
 * @author Marcelo Aguiar Rodrigues
 */
@Service
public class CacheInterceptorService {

    /**
     * Checks if the request is in cache. If true then returns the cached response, otherwise
     * continues the request normally and signals to create the cache for this request.
     *
     * @param cacheName   Cache name provided
     * @param timeToLive  How much time the cache will live (0 or less to live forever)
     * @param headers     List of headers that when present signal that the request should be cached
     * @param queryParams List of queryParams that when present signal that the request should be cached
     */
    public void cacheInterceptor(String cacheName, Long timeToLive, List<String> headers, List<String> queryParams) {

        Helper helper = new HelperImpl();

        RedissonClient redisson = (RedissonClient) BeanManager.getBean(RedissonClient.class);

        RequestContext context = RequestContext.getCurrentContext();

        if (shouldCache(context, headers, queryParams)) {
            RBucket<ApiResponse> rBucket = redisson.getBucket(createCacheKey(context, cacheName, headers, queryParams));

            if (rBucket.get() == null) {
                context.put(CACHE_BUCKET, rBucket);
                context.put(CACHE_TIME_TO_LIVE, timeToLive);
            } else {
                ApiResponse response = rBucket.get();

                helper.call().response().header().addAll(response.getHeaders());
                helper.call().response().setBody(response.getBody());
                helper.call().response().setStatus(response.getStatus());
            }
        }
    }

    /**
     * Clears a cache if it exists
     *
     * @param cacheName Cache name provided
     */
    public void cacheClearInterceptor(String cacheName) {

        RedissonClient redisson = (RedissonClient) BeanManager.getBean(RedissonClient.class);

        RequestContext context = RequestContext.getCurrentContext();

        redisson.getKeys().deleteByPattern(createDeleteCacheKey(context, cacheName));
    }

    /*
     * Creates the cache key
     */
    private String createCacheKey(RequestContext context, String cacheName, List<String> headers, List<String> queryParams) {

        String apiId = (String) context.get("api-id");
        String apiName = (String) context.get("api-name");

        StringBuilder headersBuilder = new StringBuilder();
        headers.forEach(s -> headersBuilder
                .append(s)
                .append("=")
                .append(context.getRequest().getHeader(s))
        );

        StringBuilder queryParamsBuilder = new StringBuilder();
        queryParams.forEach(s -> queryParamsBuilder
                .append(s)
                .append("=")
                .append(context.getRequestQueryParams().get(s))
        );

        return apiId + "-" + apiName + ":" +
                cacheName + ":" +
                context.getRequest().getRequestURL().toString() + ":" +
                "queryParams" + queryParamsBuilder.toString() + ":" +
                "headers" + headersBuilder.toString();
    }

    /*
     * Creates a cache key to delete the cache
     */
    private String createDeleteCacheKey(RequestContext context, String cacheName) {

        return context.get("api-id") + "-" +
                context.get("api-name") + ":" +
                cacheName + ":*";
    }

    /*
     * Defines if the cache should be written
     */
    private boolean shouldCache(RequestContext context, List<String> headers, List<String> queryParams) {

        Map<String, List<String>> requestQueryParams = (context.getRequestQueryParams() != null) ? context.getRequestQueryParams() : new HashMap<>();

        boolean queries = Lists.newArrayList(requestQueryParams.keySet()).containsAll(queryParams);

        boolean header = (context.getRequest().getHeaderNames() != null) && Collections.list(context.getRequest().getHeaderNames()).containsAll(headers);

        return header && queries;
    }

}
