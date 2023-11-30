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
package br.com.conductor.heimdall.gateway.filter;

import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.POST_TYPE;

import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.nio.charset.StandardCharsets;
import java.util.Collection;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

import javax.servlet.http.HttpServletResponse;

import org.apache.commons.lang.exception.ExceptionUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.util.StreamUtils;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;

import br.com.conductor.heimdall.core.util.Constants;
import br.com.conductor.heimdall.core.util.ContentTypeUtils;
import br.com.conductor.heimdall.core.util.UrlUtil;
import br.com.conductor.heimdall.gateway.trace.FilterDetail;
import br.com.conductor.heimdall.gateway.trace.RequestResponseParser;
import br.com.conductor.heimdall.gateway.trace.StackTraceImpl;
import br.com.conductor.heimdall.gateway.trace.TraceContextHolder;
import br.com.conductor.heimdall.middleware.spec.Helper;
import br.com.twsoftware.alfred.object.Objeto;
import lombok.Cleanup;

/**
 * Logs the response to the trace
 *
 * @author Marcelo Aguiar Rodrigues
 */
@Component
public class LogResponseFilter extends ZuulFilter {

    private FilterDetail detail = new FilterDetail();
    @Autowired
    private Helper helper;

    @Override
    public int filterOrder() {

        return 102;
    }

    @Override
    public String filterType() {

        return POST_TYPE;
    }

    @Override
    public boolean shouldFilter() {

        return true;
    }

    @Override
    public Object run() {
        long startTime = System.currentTimeMillis();
        try {
            execute();
            detail.setStatus(Constants.SUCCESS);
        } catch (Throwable e) {
            detail.setStatus(Constants.FAILED);
            TraceContextHolder.getInstance().getActualTrace().setStackTrace(new StackTraceImpl(e.getClass().getName(), e.getMessage(), ExceptionUtils.getStackTrace(e)));
        } finally {
            long endTime = System.currentTimeMillis();

            long duration = (endTime - startTime);

            detail.setName(this.getClass().getSimpleName());
            detail.setTimeInMillisRun(duration);
            TraceContextHolder.getInstance().getActualTrace().addFilter(detail);
        }
        return null;
    }

    private void execute() throws Throwable {
        RequestContext ctx = RequestContext.getCurrentContext();


        RequestResponseParser r = new RequestResponseParser();
        r.setUri(UrlUtil.getCurrentUrl(ctx.getRequest()));

        Map<String, String> headers = getResponseHeaders(ctx);
        r.setHeaders(headers);

        String content = headers.get(HttpHeaders.CONTENT_TYPE);

        // if the content type is not defined by api server then permit to read the body. Prevent NPE
        if (Objeto.isBlank(content)) content = "";

        String[] types = content.split(";");

        if (!ContentTypeUtils.belongsToBlackList(types)) {
        	@Cleanup
            InputStream stream = ctx.getResponseDataStream();
            String body;

            body = StreamUtils.copyToString(stream, StandardCharsets.UTF_8);

            if (body.isEmpty() && helper.call().response().getBody() != null) {
            	
            	body = helper.call().response().getBody();            	
            }

            if (Objects.nonNull(body) && !body.isEmpty()) {
            	
            	r.setBody(body);
            }
            ctx.setResponseDataStream(new ByteArrayInputStream(body.getBytes(StandardCharsets.UTF_8)));
        }
        TraceContextHolder.getInstance().getActualTrace().setResponse(r);
    }

    private Map<String, String> getResponseHeaders(RequestContext context) {
        Map<String, String> headers = new HashMap<>();

        final HttpServletResponse response = context.getResponse();
        
        context.getZuulResponseHeaders().stream().forEach(pair -> headers.put(pair.first(), pair.second()));
        
        final Collection<String> headerNames = response.getHeaderNames();

        headerNames.forEach(s -> headers.putIfAbsent(s, response.getHeader(s)));       

        return headers;
    }

}
