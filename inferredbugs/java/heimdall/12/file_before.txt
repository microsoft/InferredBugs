
package br.com.conductor.heimdall.core.service;

/*-
 * =========================LICENSE_START==================================
 * heimdall-core
 * ========================================================================
 * Copyright (C) 2018 Conductor Tecnologia SA
 * ========================================================================
 * Licensed under the Apache License, Version 2.0 (the "License")
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

import static br.com.conductor.heimdall.core.exception.ExceptionMessage.GLOBAL_RESOURCE_NOT_FOUND;
import static br.com.conductor.heimdall.core.exception.ExceptionMessage.ONLY_ONE_OPERATION_PER_RESOURCE;
import static br.com.conductor.heimdall.core.exception.ExceptionMessage.OPERATION_CANT_HAVE_SINGLE_WILDCARD;
import static br.com.conductor.heimdall.core.exception.ExceptionMessage.OPERATION_CANT_HAVE_DOUBLE_WILDCARD_NOT_AT_THE_END;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

import br.com.conductor.heimdall.core.entity.Api;
import br.com.conductor.heimdall.core.util.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Example;
import org.springframework.data.domain.ExampleMatcher;
import org.springframework.data.domain.ExampleMatcher.StringMatcher;
import org.springframework.data.domain.Page;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import br.com.conductor.heimdall.core.converter.GenericConverter;
import br.com.conductor.heimdall.core.dto.OperationDTO;
import br.com.conductor.heimdall.core.dto.PageDTO;
import br.com.conductor.heimdall.core.dto.PageableDTO;
import br.com.conductor.heimdall.core.dto.page.OperationPage;
import br.com.conductor.heimdall.core.entity.Operation;
import br.com.conductor.heimdall.core.entity.Resource;
import br.com.conductor.heimdall.core.exception.HeimdallException;
import br.com.conductor.heimdall.core.repository.OperationRepository;
import br.com.conductor.heimdall.core.repository.ResourceRepository;
import br.com.conductor.heimdall.core.service.amqp.AMQPCacheService;
import br.com.conductor.heimdall.core.service.amqp.AMQPRouteService;
import br.com.conductor.heimdall.core.util.ConstantsCache;
import br.com.conductor.heimdall.core.util.Pageable;

/**
 * This class provides methods to create, read, update and delete a {@link Operation} resource.
 * 
 * @author Filipe Germano
 * @author Marcelo Aguiar Rodrigues
 *
 */
@Service
public class OperationService {

     @Autowired
     private OperationRepository operationRepository;

     @Autowired
     private ResourceRepository resourceRepository;

     @Autowired
     private InterceptorService interceptorService;

     @Autowired
     private ApiService apiService;

     @Autowired
     private AMQPRouteService amqpRoute;

     @Autowired
     private AMQPCacheService amqpCacheService;

     /**
      * Finds a {@link Operation} by its Id, {@link Resource} Id and {@link br.com.conductor.heimdall.core.entity.Api} Id.
      * 
      * @param  apiId						The {@link br.com.conductor.heimdall.core.entity.Api} Id
      * @param 	resourceId					The {@link Resource} Id
      * @param 	operationId					The {@link Operation} Id
      * @return								The {@link Operation} found
      */
     @Transactional(readOnly = true)
     public Operation find(Long apiId, Long resourceId, Long operationId) {
          
          Operation operation = operationRepository.findByResourceApiIdAndResourceIdAndId(apiId, resourceId, operationId);      
          HeimdallException.checkThrow(operation == null, GLOBAL_RESOURCE_NOT_FOUND);
                              
          return operation;
     }
     
     /**
      * Generates a paged list of {@link Operation} from a request.
      * 
      * @param  apiId						The {@link br.com.conductor.heimdall.core.entity.Api} Id
      * @param 	resourceId					The {@link Resource} Id
      * @param 	operationDTO				The {@link OperationDTO}
      * @param 	pageableDTO					The {@link PageableDTO}
      * @return								The paged {@link Operation} list as a {@link OperationPage} object
      */
     @Transactional(readOnly = true)
     public OperationPage list(Long apiId, Long resourceId, OperationDTO operationDTO, PageableDTO pageableDTO) {

          Resource resource = resourceRepository.findByApiIdAndId(apiId, resourceId);
          HeimdallException.checkThrow(resource == null, GLOBAL_RESOURCE_NOT_FOUND);

          Operation operation = GenericConverter.mapper(operationDTO, Operation.class);
          operation.setResource(resource);
          
          Example<Operation> example = Example.of(operation, ExampleMatcher.matching().withIgnorePaths("resource.api").withIgnoreCase().withStringMatcher(StringMatcher.CONTAINING));
          
          Pageable pageable = Pageable.setPageable(pageableDTO.getOffset(), pageableDTO.getLimit());
          Page<Operation> page = operationRepository.findAll(example, pageable);

          return new OperationPage(PageDTO.build(page));
     }

     /**
      * Generates a list of {@link Operation} from a request.
      * 
      * @param  apiId						The {@link br.com.conductor.heimdall.core.entity.Api} Id
      * @param 	resourceId					The {@link Resource} Id
      * @param 	operationDTO				The {@link OperationDTO}
      * @return								The list of {@link Operation}
      */
     @Transactional(readOnly = true)
     public List<Operation> list(Long apiId, Long resourceId, OperationDTO operationDTO) {
          
          Resource resource = resourceRepository.findByApiIdAndId(apiId, resourceId);
          HeimdallException.checkThrow(resource == null, GLOBAL_RESOURCE_NOT_FOUND);
          
          Operation operation = GenericConverter.mapper(operationDTO, Operation.class);
          operation.setResource(resource);
          
          Example<Operation> example = Example.of(operation, ExampleMatcher.matching().withIgnorePaths("resource.api").withIgnoreCase().withStringMatcher(StringMatcher.CONTAINING));

          return operationRepository.findAll(example);
     }

     /**
      * Lists all {@link Operation} from one {@link Api}
      *
      * @param apiId The {@link Api} Id
      * @return The complete list of all {@link Operation} from the {@link Api}
      */
     @Transactional(readOnly = true)
     public List<Operation> list(final Long apiId) {
          final Api api = apiService.find(apiId);

          final List<Operation> operations = new ArrayList<>();

          final OperationDTO operationDTO = new OperationDTO();

          api.getResources().forEach(resource -> operations.addAll(this.list(apiId, resource.getId(), operationDTO)));

          if (!operations.isEmpty()) {
               return operations.stream().map(op -> new Operation(op.getId(), op.getMethod(), op.getPath(), null, null)).collect(Collectors.toList());
          }

          return new ArrayList<>();
     }
     
     /**
      * Saves a {@link Operation} to the repository.
      * 
      * @param  apiId						The {@link br.com.conductor.heimdall.core.entity.Api} Id
      * @param 	resourceId					The {@link Resource} Id
      * @param 	operationDTO				The {@link OperationDTO}
      * @return								The saved {@link Operation}
      */
     @Transactional
     public Operation save(Long apiId, Long resourceId, OperationDTO operationDTO) {

          Resource resource = resourceRepository.findByApiIdAndId(apiId, resourceId);
          HeimdallException.checkThrow(resource == null, GLOBAL_RESOURCE_NOT_FOUND);
                    
          Operation resData = operationRepository.findByResourceApiIdAndMethodAndPath(apiId, operationDTO.getMethod(), operationDTO.getPath());
          HeimdallException.checkThrow(resData != null &&
                  Objects.equals(resData.getResource().getId(), resource.getId()), ONLY_ONE_OPERATION_PER_RESOURCE);

          Operation operation = GenericConverter.mapper(operationDTO, Operation.class);
          operation.setResource(resource);
          operation.setPath(StringUtils.removeMultipleSlashes(operation.getPath()));

          HeimdallException.checkThrow(validateSingleWildCardOperationPath(operation), OPERATION_CANT_HAVE_SINGLE_WILDCARD);
          HeimdallException.checkThrow(validateDoubleWildCardOperationPath(operation), OPERATION_CANT_HAVE_DOUBLE_WILDCARD_NOT_AT_THE_END);

          operation = operationRepository.save(operation);
          
          amqpRoute.dispatchRoutes();
          
          return operation;
     }

     /**
      * Saves a {@link Operation} to the repository.
      *
      * @param  apiId						The {@link br.com.conductor.heimdall.core.entity.Api} Id
      * @param 	resourceId					The {@link Resource} Id
      * @return								The saved {@link Operation}
      */
     @Transactional
     public Operation save(Long apiId, Long resourceId, Operation operation) {

          Resource resource = resourceRepository.findByApiIdAndId(apiId, resourceId);
          HeimdallException.checkThrow(resource  == null, GLOBAL_RESOURCE_NOT_FOUND);

          Operation resData = operationRepository.findByResourceApiIdAndMethodAndPath(apiId, operation.getMethod(), operation.getPath());
          HeimdallException.checkThrow(resData != null &&
                  Objects.equals(resData.getResource().getId(), resource.getId()), ONLY_ONE_OPERATION_PER_RESOURCE);

          operation.setResource(resource);
          operation.setPath(StringUtils.removeMultipleSlashes(operation.getPath()));

          HeimdallException.checkThrow(validateSingleWildCardOperationPath(operation), OPERATION_CANT_HAVE_SINGLE_WILDCARD);
          HeimdallException.checkThrow(validateDoubleWildCardOperationPath(operation), OPERATION_CANT_HAVE_DOUBLE_WILDCARD_NOT_AT_THE_END);

          operation = operationRepository.save(operation);

          amqpRoute.dispatchRoutes();

          return operation;
     }

     /**
      * Updates a {@link Operation} by its Id, {@link br.com.conductor.heimdall.core.entity.Api} Id, {@link Resource} Id and {@link OperationDTO}.
      * 
      * @param  apiId						The {@link br.com.conductor.heimdall.core.entity.Api} Id
      * @param 	resourceId					The {@link Resource} Id
      * @param 	operationId					The {@link Operation} Id
      * @param 	operationDTO				The {@link OperationDTO}
      * @return								The updated {@link Operation}
      */
     @Transactional
     public Operation update(Long apiId, Long resourceId, Long operationId, OperationDTO operationDTO) {

          Operation operation = operationRepository.findByResourceApiIdAndResourceIdAndId(apiId, resourceId, operationId);
          HeimdallException.checkThrow(operation == null, GLOBAL_RESOURCE_NOT_FOUND);
          
          Operation resData = operationRepository.findByResourceApiIdAndMethodAndPath(apiId, operationDTO.getMethod(), operationDTO.getPath());
          HeimdallException.checkThrow(resData != null &&
                  resData.getResource().getId().equals(operation.getResource().getId()) &&
                  !resData.getId().equals(operation.getId()), ONLY_ONE_OPERATION_PER_RESOURCE);
          
          operation = GenericConverter.mapper(operationDTO, operation);
          operation.setPath(StringUtils.removeMultipleSlashes(operation.getPath()));

          HeimdallException.checkThrow(validateSingleWildCardOperationPath(operation), OPERATION_CANT_HAVE_SINGLE_WILDCARD);
          HeimdallException.checkThrow(validateDoubleWildCardOperationPath(operation), OPERATION_CANT_HAVE_DOUBLE_WILDCARD_NOT_AT_THE_END);
          
          operation = operationRepository.save(operation);
          
          amqpRoute.dispatchRoutes();
          
          amqpCacheService.dispatchClean(ConstantsCache.OPERATION_ACTIVE_FROM_ENDPOINT, operation.getResource().getApi().getBasePath() + operation.getPath());
          
          return operation;
     }
     
     /**
      * Deletes a {@link Operation} by its Id, {@link Resource} Id and {@link br.com.conductor.heimdall.core.entity.Api} Id.
      * 
      * @param  apiId						The {@link br.com.conductor.heimdall.core.entity.Api} Id
      * @param 	resourceId					The {@link Resource} Id
      * @param 	operationId					The {@link Operation} Id
      */
     @Transactional
     public void delete(Long apiId, Long resourceId, Long operationId) {

          Operation operation = operationRepository.findByResourceApiIdAndResourceIdAndId(apiId, resourceId, operationId);
          HeimdallException.checkThrow(operation == null, GLOBAL_RESOURCE_NOT_FOUND);

          // Deletes all interceptors attached to the Operation
          interceptorService.deleteAllfromOperation(operationId);

          operationRepository.delete(operation.getId());
          amqpCacheService.dispatchClean(ConstantsCache.OPERATION_ACTIVE_FROM_ENDPOINT, operation.getResource().getApi().getBasePath() + operation.getPath());
          
          
          amqpRoute.dispatchRoutes();
     }

     /**
      * Deletes all Operations from a Resource
      *
      * @param apiId      Api with the Resource
      * @param resourceId Resource with the Operations
      */
     @Transactional
     public void deleteAllfromResource(Long apiId, Long resourceId) {
          List<Operation> operations = operationRepository.findByResourceApiIdAndResourceId(apiId, resourceId);
          operations.forEach(operation -> this.delete(apiId, resourceId, operation.getId()));
     }

     /*
      * A Operation can not have a single wild card at any point in it.
      * 
      * @return  true when the path of the operation contains a single wild card, false otherwise
      */
     private boolean validateSingleWildCardOperationPath(Operation operation) {
         
          return Arrays.asList(operation.getPath().split("/")).contains("*");
     }
     
     /*
      * A Operation can have a one double wild card that must to be at the end of it, not at any other point.
      * 
      * @return true when the path has more than one double wild card or one not at the end, false otherwise
      */
     private boolean validateDoubleWildCardOperationPath(Operation operation) {
         List<String> path = Arrays.asList(operation.getPath().split("/"));
                   
         if (path.contains("**"))
        	 return !operation.getPath().endsWith("**") || !(path.stream().filter(o -> o.equals("**")).count() == 1);              
         else 
       	 	 return false;
    }

}
