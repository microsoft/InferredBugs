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

import br.com.conductor.heimdall.core.util.Constants;
import br.com.conductor.heimdall.core.util.DigestUtils;
import br.com.conductor.heimdall.core.util.UrlUtil;
import br.com.conductor.heimdall.gateway.filter.helper.HelperImpl;
import br.com.conductor.heimdall.gateway.trace.FilterDetail;
import br.com.conductor.heimdall.gateway.trace.RequestResponseParser;
import br.com.conductor.heimdall.gateway.trace.StackTraceImpl;
import br.com.conductor.heimdall.gateway.trace.TraceContextHolder;
import br.com.conductor.heimdall.middleware.spec.Helper;
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.apache.commons.lang.exception.ExceptionUtils;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import java.util.Enumeration;
import java.util.HashMap;
import java.util.Map;

import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.PRE_TYPE;

/**
 * Logs the request to the trace
 *
 * @author Marcelo Aguiar Rodrigues
 */
@Component
public class LogRequestFilter extends ZuulFilter {

    private FilterDetail detail = new FilterDetail();

    @Override
    public int filterOrder() {

        return 99;
    }

    @Override
    public String filterType() {

        return PRE_TYPE;
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

    private void execute() {
        Helper helper = new HelperImpl();

        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        RequestResponseParser r = new RequestResponseParser();
        r.setHeaders(getRequestHeadersInfo(request));
        r.setBody(helper.call().request().getBody());
        r.setUri(UrlUtil.getCurrentUrl(request));

        TraceContextHolder.getInstance().getActualTrace().setRequest(r);
    }

    private Map<String, String> getRequestHeadersInfo(HttpServletRequest request) {

        HashMap<String, String> map = new HashMap<>();
        Enumeration<String> headerNames = request.getHeaderNames();
        while (headerNames.hasMoreElements()) {

            String key = headerNames.nextElement();

            String value;
            if ("access_token".equals(key) || "client_id".equals(key)) {

                value = DigestUtils.digestMD5(request.getHeader(key));
            } else {

                value = request.getHeader(key);
            }

            map.put(key, value);
        }

        return map;
    }

}