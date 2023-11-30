
package br.com.conductor.heimdall.gateway.filter;

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

import static br.com.conductor.heimdall.core.util.Constants.INTERRUPT;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.FORWARD_LOCATION_PREFIX;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.FORWARD_TO_KEY;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.HTTPS_PORT;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.HTTPS_SCHEME;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.HTTP_PORT;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.HTTP_SCHEME;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.PROXY_KEY;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.REQUEST_URI_KEY;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.RETRYABLE_KEY;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.SERVICE_HEADER;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.SERVICE_ID_HEADER;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.SERVICE_ID_KEY;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.X_FORWARDED_FOR_HEADER;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.X_FORWARDED_HOST_HEADER;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.X_FORWARDED_PORT_HEADER;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.X_FORWARDED_PROTO_HEADER;

import java.util.List;
import java.util.Map.Entry;
import java.util.Optional;

import javax.servlet.http.HttpServletRequest;

import org.springframework.cloud.netflix.zuul.filters.ProxyRequestHelper;
import org.springframework.cloud.netflix.zuul.filters.Route;
import org.springframework.cloud.netflix.zuul.filters.ZuulProperties;
import org.springframework.cloud.netflix.zuul.filters.ZuulProperties.ZuulRoute;
import org.springframework.cloud.netflix.zuul.filters.pre.PreDecorationFilter;
import org.springframework.cloud.netflix.zuul.util.RequestUtils;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.util.AntPathMatcher;
import org.springframework.util.PathMatcher;
import org.springframework.util.StringUtils;
import org.springframework.web.util.UrlPathHelper;

import com.netflix.zuul.context.RequestContext;

import br.com.conductor.heimdall.core.entity.Environment;
import br.com.conductor.heimdall.core.entity.Operation;
import br.com.conductor.heimdall.core.repository.OperationRepository;
import br.com.conductor.heimdall.core.util.Constants;
import br.com.conductor.heimdall.core.util.ConstantsPath;
import br.com.conductor.heimdall.core.util.UrlUtil;
import br.com.conductor.heimdall.gateway.trace.FilterDetail;
import br.com.conductor.heimdall.gateway.trace.TraceContextHolder;
import br.com.conductor.heimdall.gateway.util.RequestHelper;
import br.com.conductor.heimdall.gateway.zuul.route.HeimdallRoute;
import br.com.conductor.heimdall.gateway.zuul.route.ProxyRouteLocator;
import br.com.twsoftware.alfred.object.Objeto;
import lombok.extern.slf4j.Slf4j;

/**
 * Extends the {@link PreDecorationFilter}.
 *
 * @author Marcos Filho
 *
 */
@Slf4j
public class HeimdallDecorationFilter extends PreDecorationFilter {

     private ProxyRouteLocator routeLocator;

     private String dispatcherServletPath;

     private String zuulServletPath;

     private ZuulProperties properties;

     private UrlPathHelper urlPathHelper = new UrlPathHelper();

     private ProxyRequestHelper proxyRequestHelper;
     
     private OperationRepository operationRepository;

     private PathMatcher pathMatcher = new AntPathMatcher();

     private RequestHelper requestHelper;
     
     private FilterDetail detail = new FilterDetail();
     
     public HeimdallDecorationFilter(ProxyRouteLocator routeLocator, String dispatcherServletPath, ZuulProperties properties, ProxyRequestHelper proxyRequestHelper, OperationRepository operationRepository, RequestHelper requestHelper) {

          super(routeLocator, dispatcherServletPath, properties, proxyRequestHelper);
          this.routeLocator = routeLocator;
          this.properties = properties;
          this.urlPathHelper.setRemoveSemicolonContent(properties.isRemoveSemicolonContent());
          this.dispatcherServletPath = dispatcherServletPath;
          this.proxyRequestHelper = proxyRequestHelper;
          this.operationRepository = operationRepository;
          this.zuulServletPath = properties.getServletPath();
          this.requestHelper = requestHelper;
     }
     
     @Override
     public boolean shouldFilter() {
          
          long startTime = System.currentTimeMillis();

          boolean should = super.shouldFilter();

          long endTime = System.currentTimeMillis();
          long duration = (endTime - startTime);

          detail.setTimeInMillisShould(duration);
          return should;
     }

     @Override
     public Object run() {
          long startTime = System.currentTimeMillis();
          try {
               process();
               detail.setStatus(Constants.SUCCESS);
          } catch (Exception e) {
               detail.setStatus(Constants.FAILED);
               throw e;
          } finally {
               long endTime = System.currentTimeMillis();
               
               long duration = (endTime - startTime);
               
               detail.setName(this.getClass().getSimpleName());
               detail.setTimeInMillisRun(duration);
               TraceContextHolder.getInstance().getActualTrace().addFilter(detail);
          }
          
          return null;
          
     }

     protected void process() {

          RequestContext ctx = RequestContext.getCurrentContext();
          final String requestURI = getPathWithoutStripSuffix(ctx.getRequest());
    
          if (pathMatcher.match(ConstantsPath.PATH_MANAGER_PATTERN, requestURI) || "/error".equals(requestURI)) {
               ctx.set(FORWARD_TO_KEY, requestURI);
               return;
          }
    
          final String method = ctx.getRequest().getMethod().toUpperCase();
          HeimdallRoute heimdallRoute = getMatchingHeimdallRoute(requestURI, method, ctx);
          
          if (ctx.getRequest().getHeader("access_token") != null) {               
               TraceContextHolder.getInstance().getActualTrace().setAccessToken(ctx.getRequest().getHeader("access_token"));
          }
          
          if (ctx.getRequest().getHeader("client_id") != null) {
               TraceContextHolder.getInstance().getActualTrace().setClientId(ctx.getRequest().getHeader("client_id"));
          }
          if (heimdallRoute != null) {
    
               if (heimdallRoute.isMethodNotAllowed()) {
                    ctx.setSendZuulResponse(false);
                    ctx.setResponseStatusCode(HttpStatus.METHOD_NOT_ALLOWED.value());
                    ctx.setResponseBody(HttpStatus.METHOD_NOT_ALLOWED.getReasonPhrase());
                    return;
               }
               
               if (heimdallRoute.getRoute() == null || heimdallRoute.getRoute().getLocation() == null) {
                    log.warn("Environment not configured for this location: {} and inbound: {}", ctx.getRequest().getRequestURL(), requestURI);
                    ctx.setSendZuulResponse(false);
                    ctx.setResponseStatusCode(HttpStatus.FORBIDDEN.value());
                    ctx.setResponseBody("Environment not configured for this inbound");
                    ctx.getResponse().setContentType(MediaType.TEXT_PLAIN_VALUE);
                    TraceContextHolder.getInstance().getActualTrace().setRequest(requestHelper.dumpRequest());
                    return;
               } 
               
               String location = heimdallRoute.getRoute().getLocation();
               ctx.put(REQUEST_URI_KEY, heimdallRoute.getRoute().getPath());
               ctx.put(PROXY_KEY, heimdallRoute.getRoute().getId());
               if (!heimdallRoute.getRoute().isCustomSensitiveHeaders()) {
                    this.proxyRequestHelper.addIgnoredHeaders(this.properties.getSensitiveHeaders().toArray(new String[0]));
               } else {
                    this.proxyRequestHelper.addIgnoredHeaders(heimdallRoute.getRoute().getSensitiveHeaders().toArray(new String[0]));
               }
    
               if (heimdallRoute.getRoute().getRetryable() != null) {
                    ctx.put(RETRYABLE_KEY, heimdallRoute.getRoute().getRetryable());
               }
    
               if (location.startsWith(HTTP_SCHEME + ":") || location.startsWith(HTTPS_SCHEME + ":")) {
                    ctx.setRouteHost(UrlUtil.getUrl(location));
                    ctx.addOriginResponseHeader(SERVICE_HEADER, location);
               } else if (location.startsWith(FORWARD_LOCATION_PREFIX)) {
                    ctx.set(FORWARD_TO_KEY, StringUtils.cleanPath(location.substring(FORWARD_LOCATION_PREFIX.length()) + heimdallRoute.getRoute().getPath()));
                    ctx.setRouteHost(null);
                    return;
               } else {
                    // set serviceId for use in filters.route.RibbonRequest
                    ctx.set(SERVICE_ID_KEY, location);
                    ctx.setRouteHost(null);
                    ctx.addOriginResponseHeader(SERVICE_ID_HEADER, location);
               }
               if (this.properties.isAddProxyHeaders()) {
                    addProxyHeaders(ctx);
                    String xforwardedfor = ctx.getRequest().getHeader(X_FORWARDED_FOR_HEADER);
                    String remoteAddr = ctx.getRequest().getRemoteAddr();
                    if (xforwardedfor == null) {
                         xforwardedfor = remoteAddr;
                    } else if (!xforwardedfor.contains(remoteAddr)) { // Prevent duplicates
                         xforwardedfor += ", " + remoteAddr;
                    }
                    ctx.addZuulRequestHeader(X_FORWARDED_FOR_HEADER, xforwardedfor);
               }
               if (this.properties.isAddHostHeader()) {
                    ctx.addZuulRequestHeader(HttpHeaders.HOST, toHostHeader(ctx.getRequest()));
               }
          } else {
               log.warn("No route found for uri: " + requestURI);
               ctx.setSendZuulResponse(false);
               ctx.setResponseStatusCode(HttpStatus.NOT_FOUND.value());
               ctx.setResponseBody(HttpStatus.NOT_FOUND.getReasonPhrase());
               ctx.getResponse().setContentType(MediaType.TEXT_PLAIN_VALUE);
               TraceContextHolder.getInstance().getActualTrace().setRequest(requestHelper.dumpRequest());
          }
     }

     protected String adjustPath(final String path) {

          String adjustedPath = path;

          if (RequestUtils.isDispatcherServletRequest() && StringUtils.hasText(this.dispatcherServletPath)) {
               if (!this.dispatcherServletPath.equals("/")) {
                    adjustedPath = path.substring(this.dispatcherServletPath.length());
                    log.debug("Stripped dispatcherServletPath");
               }
          } else if (RequestUtils.isZuulServletRequest()) {
               if (StringUtils.hasText(this.zuulServletPath) && !this.zuulServletPath.equals("/")) {
                    adjustedPath = path.substring(this.zuulServletPath.length());
                    log.debug("Stripped zuulServletPath");
               }
          } else {
               // do nothing
          }

          log.debug("adjustedPath=" + adjustedPath);
          return adjustedPath;
     }

     protected HeimdallRoute getMatchingHeimdallRoute(String requestURI, String method, RequestContext ctx) {

          boolean auxMatch = false;
          for (Entry<String, ZuulRoute> entry : routeLocator.getAtomicRoutes().get().entrySet()) {
               if (Objeto.notBlank(entry.getKey())) {
                    String pattern = entry.getKey();
                    if (this.pathMatcher.match(pattern, requestURI)) {
                          
                         auxMatch = true;
                         List<Operation> operations = operationRepository.findByEndPoint(pattern);
                         Operation operation = null;
                         if (Objeto.notBlank(operations)) {
                              operation = operations.stream().filter(o -> method.equals(o.getMethod().name().toUpperCase())).findFirst().orElse(null);
                         }

                         if (Objeto.notBlank(operation)) {
                              ZuulRoute zuulRoute = entry.getValue();

                              String basePath = operation.getResource().getApi().getBasePath();
                              requestURI = org.apache.commons.lang.StringUtils.removeStart(requestURI, basePath);
                              ctx.put("pattern", org.apache.commons.lang.StringUtils.removeStart(pattern, basePath));
                              
                              List<Environment> environments = operation.getResource().getApi().getEnvironments();
                              
                              String location = null;
                              if (Objeto.notBlank(environments)) {
                                   
                                   String host = ctx.getRequest().getHeader("Host");                                   
                                   if (Objeto.isBlank(host)) {

                                        host = ctx.getRequest().getHeader("host");
                                   }
                                   
                                   Optional<Environment> environment = Optional.ofNullable(null);
                                   if (Objeto.notBlank(host)) {
                                        String tempHost = host;
                                        environment = environments.stream().filter(e -> e.getInboundURL().toLowerCase().contains(tempHost.toLowerCase())).findFirst();                                        
                                   } else {
                                        environment = environments.stream().filter(e -> ctx.getRequest().getRequestURL().toString().toLowerCase().contains(e.getInboundURL().toLowerCase())).findFirst();
                                   }
                                   
                                   if (environment.isPresent()) {
                                        location = environment.get().getOutboundURL();
                                        ctx.put("environmentVariables", environment.get().getVariables());
                                   }
                              }                              
                              Route route = new Route(zuulRoute.getId(), requestURI, location, "", zuulRoute.getRetryable() != null ? zuulRoute.getRetryable() : false, zuulRoute.isCustomSensitiveHeaders() ? zuulRoute.getSensitiveHeaders() : null);

                              TraceContextHolder traceContextHolder = TraceContextHolder.getInstance();
                              
                              traceContextHolder.getActualTrace().setApiId(operation.getResource().getApi().getId());
                              traceContextHolder.getActualTrace().setApiName(operation.getResource().getApi().getName());
                              traceContextHolder.getActualTrace().setResourceId(operation.getResource().getId());
                              traceContextHolder.getActualTrace().setOperationId(operation.getId());
                              traceContextHolder.getActualTrace().setPattern(operation.getPath());
                              
                              return new HeimdallRoute(pattern, route, false);
                         } else {
                              
                              ctx.put(INTERRUPT, true);
                         }
                    }
               }
          }

          if (auxMatch) {
               return new HeimdallRoute().methodNotAllowed();
          }

          return null;
     }

     protected String getPathWithoutStripSuffix(HttpServletRequest request) {

          String URI = this.urlPathHelper.getPathWithinApplication(request);

          if (URI.endsWith(ConstantsPath.PATH_ROOT)) {
               URI = org.apache.commons.lang.StringUtils.removeEnd(URI, ConstantsPath.PATH_ROOT);
          }

          URI = adjustPath(URI);

          return URI;
     }

     protected void addProxyHeaders(RequestContext ctx) {

          HttpServletRequest request = ctx.getRequest();
          String host = toHostHeader(request);
          String port = String.valueOf(request.getServerPort());
          String proto = request.getScheme();
          if (hasHeader(request, X_FORWARDED_HOST_HEADER)) {
               host = request.getHeader(X_FORWARDED_HOST_HEADER) + "," + host;
          }
          if (!hasHeader(request, X_FORWARDED_PORT_HEADER)) {
               if (hasHeader(request, X_FORWARDED_PROTO_HEADER)) {
                    StringBuilder builder = new StringBuilder();
                    for (String previous : StringUtils.commaDelimitedListToStringArray(request.getHeader(X_FORWARDED_PROTO_HEADER))) {
                         if (builder.length() > 0) {
                              builder.append(",");
                         }
                         builder.append(HTTPS_SCHEME.equals(previous) ? HTTPS_PORT : HTTP_PORT);
                    }
                    builder.append(",").append(port);
                    port = builder.toString();
               }
          } else {
               port = request.getHeader(X_FORWARDED_PORT_HEADER) + "," + port;
          }
          if (hasHeader(request, X_FORWARDED_PROTO_HEADER)) {
               proto = request.getHeader(X_FORWARDED_PROTO_HEADER) + "," + proto;
          }
          ctx.addZuulRequestHeader(X_FORWARDED_HOST_HEADER, host);
          ctx.addZuulRequestHeader(X_FORWARDED_PORT_HEADER, port);
          ctx.addZuulRequestHeader(X_FORWARDED_PROTO_HEADER, proto);
     }

     protected boolean hasHeader(HttpServletRequest request, String name) {

          return StringUtils.hasLength(request.getHeader(name));
     }

     protected String toHostHeader(HttpServletRequest request) {

          int port = request.getServerPort();
          if ((port == HTTP_PORT && HTTP_SCHEME.equals(request.getScheme())) || (port == HTTPS_PORT && HTTPS_SCHEME.equals(request.getScheme()))) {
               return request.getServerName();
          } else {
               return request.getServerName() + ":" + port;
          }
     }
}
