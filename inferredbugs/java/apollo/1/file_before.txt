package io.logz.apollo.status;

import io.logz.apollo.dao.DeployableVersionDao;
import io.logz.apollo.dao.DeploymentDao;
import io.logz.apollo.dao.EnvironmentDao;
import io.logz.apollo.dao.ServiceDao;
import io.logz.apollo.models.DeployableVersion;
import io.logz.apollo.models.Deployment;
import io.logz.apollo.models.Service;
import javax.inject.Inject;
import java.time.LocalDate;
import java.time.ZoneId;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;
import java.util.concurrent.TimeUnit;

import static java.util.Objects.requireNonNull;

public class ServiceStatusHandler {
    private final DeploymentDao deploymentDao;
    private final DeployableVersionDao deployableVersionDao;
    private final EnvironmentDao environmentDao;
    private final ServiceDao serviceDao;
    private static final int MIN_DAYS_OF_UNDEPLOYED_SERVICES = 21;

    @Inject
    public ServiceStatusHandler(DeploymentDao deploymentDao, DeployableVersionDao deployableVersionDao, EnvironmentDao environmentDao, ServiceDao serviceDao) {
        this.deploymentDao = requireNonNull(deploymentDao);
        this.deployableVersionDao = requireNonNull(deployableVersionDao);
        this.environmentDao = requireNonNull(environmentDao);
        this.serviceDao = requireNonNull(serviceDao);
    }

    public List<Integer> getUndeployedServices() {
        List<Integer> productionEnvironmentsIds = getProductionEnvironmentsIds();
        List<Integer> undeployedServices = new ArrayList<>();
        productionEnvironmentsIds.forEach(environmentId -> undeployedServices.addAll(getUndeployedServicesByEnvironmentId(environmentId)));
        return undeployedServices;
    }

    private List<Integer> getUndeployedServicesByEnvironmentId(int environmentId) {
        List<Service> services = serviceDao.getAllServices();
        List<Integer> undeployedServicesIds = new ArrayList<>();
        services.forEach(service -> {
            if(isServiceUndeployed(environmentId, service.getId())) {
                undeployedServicesIds.add(service.getId());
            }
        });
        return undeployedServicesIds;
    }

    private List<Integer> getProductionEnvironmentsIds() {
        return environmentDao.getProductionEnvironmentsIds();
    }

    private boolean isServiceUndeployed(int environmentId, int serviceId){
        Date latestDeploymentDate = getLatestDeploymentDateByServiceIdInEnvironment(environmentId, serviceId);
        Date latestUpdatedDate = getLatestDeployableVersionDateByServiceId(serviceId);

        if(latestDeploymentDate != null) {
            long diffBetweenDatesInDays = TimeUnit.DAYS.convert(Math.abs(latestDeploymentDate.getTime() - latestUpdatedDate.getTime()), TimeUnit.MILLISECONDS);
            return diffBetweenDatesInDays > MIN_DAYS_OF_UNDEPLOYED_SERVICES ? true : false;
        }
        if(latestUpdatedDate != null) {
            Date durationSinceLatestUpdate = Date.from(LocalDate.now().minusDays((long) MIN_DAYS_OF_UNDEPLOYED_SERVICES).atStartOfDay(ZoneId.systemDefault()).toInstant());
            if(latestUpdatedDate.compareTo(durationSinceLatestUpdate) < 0) {
                return true;
            }
        }
        return false;
    }

    private Date getLatestDeploymentDateByServiceIdInEnvironment(int environmentId, int serviceId) {
        Deployment deployment = deploymentDao.getLatestDeploymentOfServiceInEnvironment(serviceId, environmentId);
        if(deployment != null){
            return deployment.getLastUpdate();
        }
        return null;
    }

    private Date getLatestDeployableVersionDateByServiceId(int serviceId) {
        DeployableVersion latestDeployableVersion = deployableVersionDao.getLatestDeployableVersionByServiceId(serviceId);
        if(latestDeployableVersion != null) {
            return latestDeployableVersion.getCommitDate();
        }
        return null;
    }
}
