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
import br.com.conductor.heimdall.gateway.failsafe.CircuitBreakerManager;
import br.com.conductor.heimdall.gateway.trace.FilterDetail;
import br.com.conductor.heimdall.gateway.trace.TraceContextHolder;
import com.netflix.zuul.context.RequestContext;
import lombok.extern.slf4j.Slf4j;
import org.apache.http.HttpHost;
import org.springframework.cloud.netflix.zuul.filters.ProxyRequestHelper;
import org.springframework.cloud.netflix.zuul.filters.ZuulProperties;
import org.springframework.cloud.netflix.zuul.filters.route.SimpleHostRoutingFilter;

import javax.servlet.http.HttpServletRequest;
import java.util.concurrent.Callable;

import static br.com.conductor.heimdall.gateway.util.ConstantsContext.OPERATION_ID;
import static br.com.conductor.heimdall.gateway.util.ConstantsContext.OPERATION_PATH;

/**
 * Creates a custom routing filter.
 *
 * @author Marcos Filho
 *
 */
@Slf4j
public class CustomHostRoutingFilter extends SimpleHostRoutingFilter {
     
     private FilterDetail detail = new FilterDetail();
     private final CircuitBreakerManager circuitBreakerManager;

     /**
      * Creates a CustomHostRoutingFilter from a {@link ProxyRequestHelper} and a {@link ZuulProperties}.
      * 
      * @param helper		{@link ProxyRequestHelper}
      * @param properties	{@link ZuulProperties}
      */
     public CustomHostRoutingFilter(ProxyRequestHelper helper, ZuulProperties properties, CircuitBreakerManager circuitBreakerManager) {
          super(helper, properties);
          this.circuitBreakerManager = circuitBreakerManager;
     }
     
     /**
      * Checks if it should filter and sets the duration time.
      */
     @Override
     public boolean shouldFilter() {
          long startTime = System.currentTimeMillis();
          
          boolean should = super.shouldFilter();
          
          long endTime = System.currentTimeMillis();
          long duration = (endTime - startTime);
          
          detail.setTimeInMillisShould(duration);
          return should;
     }
     
	/**
	 * Runs the custom routing filter.
	 */
	@Override
	public Object run() {
		long startTime = System.currentTimeMillis();

		RequestContext context = RequestContext.getCurrentContext();
		HttpServletRequest request = context.getRequest();
		HttpHost httpHost = new HttpHost(
				context.getRouteHost().getHost(),
				context.getRouteHost().getPort(),
				context.getRouteHost().getProtocol());

		Long operationId = (Long) context.get(OPERATION_ID);
		String operationPath = (String) context.get(OPERATION_PATH);

		try {
			Callable<Object> callable = super::run;
			Object obj = circuitBreakerManager.failsafe(callable, operationId, operationPath);
			detail.setStatus(Constants.SUCCESS);
			return obj;
		} catch (Exception e) {
			detail.setStatus(Constants.FAILED);
			log.error("Exception: {} - Message: {} - during routing request to (hostPath + uri): {} - Verb: {} - HostName: {} - Port: {} - SchemeName: {}",
					e.getClass().getName(), 
					e.getMessage(), 
					request.getRequestURI(),
					request.getMethod().toUpperCase(),
					httpHost.getHostName(),
					httpHost.getPort(), 
					httpHost.getSchemeName());
			throw e;
		} finally {
			long endTime = System.currentTimeMillis();

			long duration = (endTime - startTime);

			detail.setName(this.getClass().getSimpleName());
			detail.setTimeInMillisRun(duration);
			TraceContextHolder.getInstance().getActualTrace().addFilter(detail);
		}
	}

}
