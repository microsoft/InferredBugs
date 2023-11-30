package io.logz.apollo;

import io.logz.apollo.clients.ApolloTestAdminClient;
import io.logz.apollo.clients.ApolloTestClient;
import io.logz.apollo.exceptions.ApolloClientException;
import io.logz.apollo.helpers.Common;
import io.logz.apollo.helpers.ModelsGenerator;
import io.logz.apollo.models.BlockerDefinition;
import io.logz.apollo.models.DeployableVersion;
import io.logz.apollo.models.Environment;
import io.logz.apollo.models.Service;
import io.logz.apollo.models.Stack;
import org.junit.Ignore;
import org.junit.Test;

import java.time.LocalDate;
import java.time.LocalTime;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

import static io.logz.apollo.helpers.ModelsGenerator.createAndSubmitBlocker;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

/**
 * Created by roiravhon on 6/5/17.
 */
public class BlockerTest {

    private final static String APOLLO_REPO_URL = "https://github.com/logzio/apollo";
    private final static String APOLLO_FAILED_STATUS_CHECKS_COMMIT = "63b79a229defab92df685dcc4e47dd35f0518ef0";
    private final static String APOLLO_VALID_STATUS_CHECKS_COMMIT = "7e01c7e4cfddeadd49d4a4e9ed575b24e9210f4b";

    @Test
    public void testEverythingBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);

        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, null);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        blocker.setActive(false);
        apolloTestAdminClient.updateBlocker(blocker);

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion);
    }

    @Test
    public void testUnconditionalBlockerWithServiceAndEnvironmentException() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);

        List<Integer> exceptionServiceIds = new ArrayList<Integer>() {{ add(service.getId()); }};
        List<Integer> exceptionEnvironmentIds = new ArrayList<Integer>() {{ add(environment.getId()); }};

        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "unconditional", getUnconditionalBlockerConfiguration(exceptionServiceIds, exceptionEnvironmentIds), null, null);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion);

        blocker.setActive(false);
        apolloTestAdminClient.updateBlocker(blocker);
    }

    @Test
    public void testUnconditionalBlockerWithEnvironmentException() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment exceptionEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);
        Environment otherEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);

        List<Integer> exceptionEnvironmentIds = new ArrayList<Integer>() {{ add(exceptionEnvironment.getId()); }};

        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "unconditional", getUnconditionalBlockerConfiguration(null, exceptionEnvironmentIds), null, service);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, otherEnvironment, service, deployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, exceptionEnvironment, service, deployableVersion);

        blocker.setActive(false);
        apolloTestAdminClient.updateBlocker(blocker);
    }

    @Test
    public void testUnconditionalBlockerWithServiceException() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service exceptionService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, exceptionService);
        Service otherService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion otherDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, otherService);

        List<Integer> exceptionServiceIds = new ArrayList<Integer>() {{ add(exceptionService.getId()); }};

        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "unconditional", getUnconditionalBlockerConfiguration(exceptionServiceIds, null), environment, null);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, otherService, otherDeployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, exceptionService, deployableVersion);

        blocker.setActive(false);
        apolloTestAdminClient.updateBlocker(blocker);
    }

    @Test
    public void testEnvironmentBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment blockedEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Environment okEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", blockedEnvironment, null);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, blockedEnvironment, service, deployableVersion)).isInstanceOf(Exception.class)
            .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, okEnvironment, service, deployableVersion);
    }

    @Test
    public void testServiceBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service blockedService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Service okService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion blockedDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, blockedService);
        DeployableVersion okDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, okService);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, blockedService);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, blockedService, blockedDeployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, okService, okDeployableVersion);
    }

    @Test
    public void testEnvironmentsStackBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment blockedEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Environment okEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Stack blockedEnvironmentsStack = ModelsGenerator.createAndSubmitEnvironmentsStack(apolloTestClient, Arrays.asList(blockedEnvironment.getId()));
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, null, blockedEnvironmentsStack);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, blockedEnvironment, service, deployableVersion)).isInstanceOf(Exception.class)
                                                                                                                                                    .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, okEnvironment, service, deployableVersion);
    }

    @Test
    public void testServicesStackBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service blockedService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Service okService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Stack blockedServicesStack = ModelsGenerator.createAndSubmitServicesStack(apolloTestClient, Arrays.asList(blockedService.getId()));
        DeployableVersion blockedDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, blockedService);
        DeployableVersion okDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, okService);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, null, blockedServicesStack);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, blockedService, blockedDeployableVersion)).isInstanceOf(Exception.class)
                                                                                                                                                    .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, okService, okDeployableVersion);
    }

    @Test
    public void testBlockByAvailabilityOnSpecificService() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        String availability = "BLOCK-ME-1";
        Environment environment = ModelsGenerator.createAndSubmitEnvironment(availability, apolloTestClient);
        Service blockedService = ModelsGenerator.createAndSubmitService(apolloTestClient);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, blockedService, null, availability);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, blockedService);
        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, blockedService, deployableVersion)).isInstanceOf(Exception.class)
                                                                                                                                                           .hasMessageContaining("Deployment is currently blocked");
    }

    @Test
    public void testBlockByAvailabilityOnAllServices() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        String availability = "BLOCK-ME-2";
        Environment environment = ModelsGenerator.createAndSubmitEnvironment(availability, apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, null, null, availability);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);
        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion)).isInstanceOf(Exception.class)
                                                                                                                                         .hasMessageContaining("Deployment is currently blocked");
    }

    @Test
    public void testNonBlockedDeploymentWithAvailabilityBlockerOnSpecificService() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        String availability = "BLOCK-ME-3";
        Environment environment = ModelsGenerator.createAndSubmitEnvironment(availability, apolloTestClient); //Environment in this availability
        Service blockedService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, blockedService, null, availability);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);
        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion);
    }

    @Test
    public void testNonBlockedDeploymentWithAnotherAvailabilityBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        String availability = "BLOCK-ME-4";
        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient); //Random availability
        Service blockedService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, blockedService, null, availability);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);
        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion);
    }

    @Test
    public void testCreatingInvalidAvailabilityBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        String availability = "BLOCK-ME-5";
        Environment environment = ModelsGenerator.createAndSubmitEnvironment(availability, apolloTestClient);
        String errorMessage = String.format("Trying to add invalid blocker. stackId - null, environmentId - %s, serviceId - null, availability - %s", environment.getId(), availability);

        assertThatThrownBy(() -> ModelsGenerator. createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", environment, null, null, availability))
                .isInstanceOf(ApolloClientException.class)
                .hasMessageContaining(errorMessage);
    }

    @Test
    public void testAddingInvalidBlockers() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Stack servicesStack = ModelsGenerator.createAndSubmitServicesStack(apolloTestClient, Arrays.asList(service.getId()));

        String errorMessage = "Trying to add invalid blocker. ";

        assertThatThrownBy(() -> ModelsGenerator. createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", null, service, servicesStack)).isInstanceOf(ApolloClientException.class)
                                                                                                                                                                   .hasMessageContaining(errorMessage + String.format("stackId - %s, environmentId - null, serviceId - %s, availability - null", servicesStack.getId(), service.getId()));

        assertThatThrownBy(() -> ModelsGenerator. createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", environment, null, servicesStack)).isInstanceOf(ApolloClientException.class)
                                                                                                                                                                   .hasMessageContaining(errorMessage + String.format("stackId - %s, environmentId - %s, serviceId - null", servicesStack.getId(), environment.getId()));

        assertThatThrownBy(() -> ModelsGenerator. createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", environment, service, servicesStack)).isInstanceOf(ApolloClientException.class)
                                                                                                                                                                .hasMessageContaining(errorMessage + String.format("stackId - %s, environmentId - %s, serviceId - %s", servicesStack.getId(), environment.getId(), service.getId()));
    }

    @Test
    public void testSpecificBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment blockedEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Environment okEnvironment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service blockedService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Service okService = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion blockedDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, blockedService);
        DeployableVersion okDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, okService);

        createAndSubmitBlocker(apolloTestAdminClient, "unconditional", "{}", blockedEnvironment, blockedService);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, blockedEnvironment, blockedService, blockedDeployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, blockedEnvironment, okService, okDeployableVersion);
        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, okEnvironment, blockedService, blockedDeployableVersion);
    }

    @Test
    public void testTimeBasedBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        DeployableVersion deployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service);

        int dayOfWeek = LocalDate.now().getDayOfWeek().getValue();
        LocalTime twoMinutesFromNow = LocalTime.now().plusMinutes(2);
        LocalTime twoMinutesBeforeNow = LocalTime.now().minusMinutes(2);
        LocalTime threeMinutesFromNow = LocalTime.now().plusMinutes(3);

        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "timebased",
                getTimeBasedBlockerJsonConfiguration(dayOfWeek, twoMinutesBeforeNow, twoMinutesFromNow),
                environment, service);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        blocker.setBlockerJsonConfiguration(getTimeBasedBlockerJsonConfiguration(dayOfWeek, twoMinutesFromNow, threeMinutesFromNow));
        apolloTestAdminClient.updateBlocker(blocker);

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion);
    }

    @Test
    @Ignore
    public void testBranchBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);

        // Create a deployable version from logzio public API repo (small public one)
        DeployableVersion deployableVersion = new DeployableVersion();
        deployableVersion.setGithubRepositoryUrl("https://github.com/logzio/public-api");
        deployableVersion.setGitCommitSha("c5265fcc8a73c9d5a79170b668c21c958b53a93e");
        deployableVersion.setServiceId(service.getId());

        deployableVersion.setId(apolloTestClient.addDeployableVersion(deployableVersion).getId());


        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "branch",
                getBranchBlockerJsonConfiguration("develop"),
                environment, service);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        blocker.setBlockerJsonConfiguration(getBranchBlockerJsonConfiguration("master"));
        apolloTestAdminClient.updateBlocker(blocker);

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, deployableVersion);
    }

    @Test
    public void testConcurrencyBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service serviceA = ModelsGenerator.createAndSubmitService(apolloTestClient);
        Service serviceB = ModelsGenerator.createAndSubmitService(apolloTestClient);

        DeployableVersion deployableVersionA = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, serviceA);
        DeployableVersion deployableVersionB = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, serviceB);

        List<Integer> excludedService = new ArrayList<>();

        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "concurrent",
                getConcurrencyBlockerJsonConfiguration(1, excludedService),
                environment, null);

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, serviceA, deployableVersionA);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, serviceB, deployableVersionB)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        excludedService.add(serviceB.getId());
        blocker.setBlockerJsonConfiguration(getConcurrencyBlockerJsonConfiguration(1, excludedService));
        apolloTestAdminClient.updateBlocker(blocker);
        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, serviceB, deployableVersionB);
    }

    @Test
    @Ignore
    public void testGHCommitStatusBlocker() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);

        DeployableVersion failedDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service,
                APOLLO_REPO_URL, APOLLO_FAILED_STATUS_CHECKS_COMMIT);
        
        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "githubCommitStatus",null, environment, service);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, failedDeployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        // Attempt to deploy a valid commit should work
        DeployableVersion validDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service,
                APOLLO_REPO_URL, APOLLO_VALID_STATUS_CHECKS_COMMIT);

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, validDeployableVersion);
    }

    @Test
    @Ignore
    public void testBlockerUserOverride() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);

        DeployableVersion failedDeployableVersion = ModelsGenerator.createAndSubmitDeployableVersion(apolloTestClient, service,
                APOLLO_REPO_URL, APOLLO_FAILED_STATUS_CHECKS_COMMIT);

        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "githubCommitStatus",null, environment, service);

        assertThatThrownBy(() -> ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, failedDeployableVersion)).isInstanceOf(Exception.class)
                .hasMessageContaining("Deployment is currently blocked");

        apolloTestAdminClient.overrideBlockerByUser(apolloTestClient.getTestUser(), blocker);

        ModelsGenerator.createAndSubmitDeployment(apolloTestClient, environment, service, failedDeployableVersion);
    }

    @Test
    public void changeBlockerActiveAttribute() throws Exception {
        ApolloTestClient apolloTestClient = Common.signupAndLogin();
        ApolloTestAdminClient apolloTestAdminClient = Common.getAndLoginApolloTestAdminClient();

        Environment environment = ModelsGenerator.createAndSubmitEnvironment(apolloTestClient);
        Service service = ModelsGenerator.createAndSubmitService(apolloTestClient);
        BlockerDefinition blocker = createAndSubmitBlocker(apolloTestAdminClient, "githubCommitStatus",null, environment, service);

        Boolean beforeActive = blocker.getActive();
        Boolean afterActive = !beforeActive;

        BlockerDefinition updatedBlocker = apolloTestClient.updateBlockerDefinitionActiveness(blocker.getId(), afterActive);
        assertThat(updatedBlocker.getActive()).isEqualTo(afterActive);
    }

    private String getTimeBasedBlockerJsonConfiguration(int dayOfWeek, LocalTime startDate, LocalTime endDate) {

        return "{\n" +
                "  \"startTimeUtc\": \"" + startDate.getHour() + ":" + startDate.getMinute() + "\",\n" +
                "  \"endTimeUtc\": \"" + endDate.getHour() + ":" + endDate.getMinute() + "\",\n" +
                "  \"daysOfTheWeek\": [\n" +
                "    " + dayOfWeek + "\n" +
                "  ]\n" +
                "}";
    }

    private String getConcurrencyBlockerJsonConfiguration(int allowedConcurrentDeployment, List<Integer> excludeServices) {
        return "{\n" +
                "  \"allowedConcurrentDeployment\": \"" + allowedConcurrentDeployment + "\",\n" +
                "  \"excludeServices\":" + excludeServices.toString() +
                "}";
    }

    private String getBranchBlockerJsonConfiguration(String branchesNames) {
        return "{\n" +
                "  \"branchName\": \"" + branchesNames + "\"\n" +
                "}";
    }

    private String getUnconditionalBlockerConfiguration(List<Integer> exceptionServiceIds, List<Integer> exceptionEnvironmentIds) {

        if (exceptionServiceIds == null && exceptionEnvironmentIds == null) {
            return "{}";
        }

        String jsonConfiguration = "{\n";

        if (exceptionServiceIds != null) {
            jsonConfiguration += "  \"exceptionServiceIds\":" + exceptionServiceIds.toString();
            if (exceptionEnvironmentIds != null) {
                jsonConfiguration += ",\n";
            }
        }

        if (exceptionEnvironmentIds != null) {
            jsonConfiguration +=     "  \"exceptionEnvironmentIds\":" + exceptionEnvironmentIds.toString();
        }

        return jsonConfiguration += "}";
    }
}
