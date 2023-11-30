package com.ctrip.apollo.portal.service;

import static org.junit.Assert.assertEquals;
import static org.mockito.Mockito.when;


import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.Mockito;
import org.mockito.Spy;
import org.mockito.runners.MockitoJUnitRunner;
import org.springframework.test.util.ReflectionTestUtils;
import org.springframework.web.client.RestTemplate;

import com.ctrip.apollo.Apollo.Env;
import com.ctrip.apollo.core.Constants;
import com.ctrip.apollo.core.dto.ClusterDTO;
import com.ctrip.apollo.core.dto.ConfigItemDTO;
import com.ctrip.apollo.core.dto.ReleaseSnapshotDTO;
import com.ctrip.apollo.core.dto.VersionDTO;
import com.ctrip.apollo.portal.api.AdminServiceAPI;
import com.ctrip.apollo.portal.constants.PortalConstants;
import com.ctrip.apollo.portal.entity.AppConfigVO;

import java.util.Arrays;

@RunWith(MockitoJUnitRunner.class)
public class ConfigServiceTest {

  @Mock
  private RestTemplate restTemplate;

  @InjectMocks
  private ConfigService configService;

  @Mock
  private ServiceLocator serviceLocator;
  @Spy
  private AdminServiceAPI.VersionAPI versionAPI;
  @Spy
  private AdminServiceAPI.ClusterAPI clusterAPI;
  @Spy
  private AdminServiceAPI.ConfigAPI configAPI;



  @Before
  public void setUp() {
    ReflectionTestUtils.setField(versionAPI, "restTemplate", restTemplate);
    ReflectionTestUtils.setField(clusterAPI, "restTemplate", restTemplate);
    ReflectionTestUtils.setField(configAPI, "restTemplate", restTemplate);

    ReflectionTestUtils.setField(versionAPI, "serviceLocator", serviceLocator);
    ReflectionTestUtils.setField(clusterAPI, "serviceLocator", serviceLocator);
    ReflectionTestUtils.setField(configAPI, "serviceLocator", serviceLocator);

    String defaultAdminService = "http://localhost:8090";
    Mockito.doReturn(defaultAdminService).when(serviceLocator).getAdminService(Env.DEV);
  }

  @Test
  public void testLoadReleaseConfig() {
    long appId = 6666;
    long versionId = 100;
    long releaseId = 11111;

    VersionDTO someVersion = assembleVersion(appId, "1.0", releaseId);
    ReleaseSnapshotDTO[] someReleaseSnapShots = assembleReleaseSnapShots();

    when(versionAPI.getVersionById(Env.DEV, versionId)).thenReturn(someVersion);
    when(configAPI.getConfigByReleaseId(Env.DEV, releaseId)).thenReturn(someReleaseSnapShots);

    AppConfigVO appConfigVO = configService.loadReleaseConfig(Env.DEV, appId, versionId);

    assertEquals(appConfigVO.getAppId(), appId);
    assertEquals(appConfigVO.getVersionId(), versionId);
    assertEquals(appConfigVO.getDefaultClusterConfigs().size(), 2);
    assertEquals(appConfigVO.getOverrideAppConfigs().size(), 2);
    assertEquals(appConfigVO.getOverrideClusterConfigs().size(), 2);
  }

  @Test
  public void testLoadReleaseConfigOnlyDefaultConfigs() {
    long appId = 6666;
    long versionId = 100;
    long releaseId = 11111;

    VersionDTO someVersion = assembleVersion(appId, "1.0", releaseId);
    ReleaseSnapshotDTO[] someReleaseSnapShots = new ReleaseSnapshotDTO[1];
    someReleaseSnapShots[0] = assembleReleaseSnapShot(11111, Constants.DEFAULT_CLUSTER_NAME,
                                                  "{\"6666.foo\":\"demo1\", \"6666.bar\":\"demo2\"}");

    when(versionAPI.getVersionById(Env.DEV, versionId)).thenReturn(someVersion);
    when(configAPI.getConfigByReleaseId(Env.DEV, releaseId)).thenReturn(someReleaseSnapShots);

    AppConfigVO appConfigVO = configService.loadReleaseConfig(Env.DEV, appId, versionId);

    assertEquals(appConfigVO.getAppId(), appId);
    assertEquals(appConfigVO.getVersionId(), versionId);
    assertEquals(appConfigVO.getDefaultClusterConfigs().size(), 2);
    assertEquals(appConfigVO.getOverrideAppConfigs().size(), 0);
    assertEquals(appConfigVO.getOverrideClusterConfigs().size(), 0);
  }

  @Test
  public void testLoadReleaseConfigDefaultConfigsAndOverrideApp() {
    long appId = 6666;
    long versionId = 100;
    long releaseId = 11111;
    VersionDTO someVersion = assembleVersion(appId, "1.0", releaseId);
    ReleaseSnapshotDTO[] someReleaseSnapShots = new ReleaseSnapshotDTO[1];
    someReleaseSnapShots[0] = assembleReleaseSnapShot(11111, Constants.DEFAULT_CLUSTER_NAME,
                                                  "{\"6666.foo\":\"demo1\", \"6666.bar\":\"demo2\", \"5555.bar\":\"demo2\", \"22.bar\":\"demo2\"}");

    when(versionAPI.getVersionById(Env.DEV, versionId)).thenReturn(someVersion);
    when(configAPI.getConfigByReleaseId(Env.DEV, releaseId)).thenReturn(someReleaseSnapShots);

    AppConfigVO appConfigVO = configService.loadReleaseConfig(Env.DEV, appId, versionId);

    assertEquals(appConfigVO.getAppId(), appId);
    assertEquals(appConfigVO.getVersionId(), versionId);
    assertEquals(appConfigVO.getDefaultClusterConfigs().size(), 2);
    assertEquals(2, appConfigVO.getOverrideAppConfigs().size());
    assertEquals(appConfigVO.getOverrideClusterConfigs().size(), 0);
  }

  @Test
  public void testLoadReleaseConfigDefaultConfigsAndOverrideCluster() {
    long appId = 6666;
    long versionId = 100;
    long releaseId = 11111;
    VersionDTO someVersion = assembleVersion(appId, "1.0", releaseId);
    ReleaseSnapshotDTO[] someReleaseSnapShots = new ReleaseSnapshotDTO[2];
    someReleaseSnapShots[0] = assembleReleaseSnapShot(11111, Constants.DEFAULT_CLUSTER_NAME,
                                                  "{\"6666.foo\":\"demo1\", \"6666.bar\":\"demo2\"}");
    someReleaseSnapShots[1] = assembleReleaseSnapShot(11112, "cluster1",
                                                  "{\"6666.foo\":\"demo1\", \"6666.bar\":\"demo2\"}");

    when(versionAPI.getVersionById(Env.DEV, versionId)).thenReturn(someVersion);
    when(configAPI.getConfigByReleaseId(Env.DEV, releaseId)).thenReturn(someReleaseSnapShots);

    AppConfigVO appConfigVO = configService.loadReleaseConfig(Env.DEV, appId, versionId);

    assertEquals(appConfigVO.getAppId(), appId);
    assertEquals(appConfigVO.getVersionId(), versionId);
    assertEquals(appConfigVO.getDefaultClusterConfigs().size(), 2);
    assertEquals(0, appConfigVO.getOverrideAppConfigs().size());
    assertEquals(1, appConfigVO.getOverrideClusterConfigs().size());
  }

  @Test
  public void testLoadLastestConfig() {
    long appId = 6666;
    ClusterDTO[] someClusters = assembleClusters();
    ConfigItemDTO[] someConfigItem = assembleConfigItems();

    when(clusterAPI.getClustersByApp(Env.DEV, appId)).thenReturn(someClusters);
    when(configAPI.getLatestConfigItemsByClusters(Env.DEV, Arrays
        .asList(Long.valueOf(100), Long.valueOf(101)))).thenReturn(someConfigItem);

    AppConfigVO appConfigVO = configService.loadLatestConfig(Env.DEV, appId);

    assertEquals(appConfigVO.getAppId(), 6666);
    assertEquals(appConfigVO.getVersionId(), PortalConstants.LASTEST_VERSION_ID);
    assertEquals(appConfigVO.getDefaultClusterConfigs().size(), 3);
    assertEquals(appConfigVO.getOverrideAppConfigs().size(), 1);
    assertEquals(appConfigVO.getOverrideClusterConfigs().size(), 1);
  }

  private VersionDTO assembleVersion(long appId, String versionName, long releaseId) {
    VersionDTO version = new VersionDTO();
    version.setAppId(appId);
    version.setName(versionName);
    version.setReleaseId(releaseId);
    return version;
  }

  private ReleaseSnapshotDTO[] assembleReleaseSnapShots() {
    ReleaseSnapshotDTO[] releaseSnapShots = new ReleaseSnapshotDTO[3];
    releaseSnapShots[0] = assembleReleaseSnapShot(11111, Constants.DEFAULT_CLUSTER_NAME,
                                                  "{\"6666.foo\":\"demo1\", \"6666.bar\":\"demo2\",\"3333.foo\":\"1008\",\"4444.bar\":\"99901\"}");
    releaseSnapShots[1] = assembleReleaseSnapShot(11111, "cluster1", "{\"6666.foo\":\"demo1\"}");
    releaseSnapShots[2] = assembleReleaseSnapShot(11111, "cluster2", "{\"6666.bar\":\"bar2222\"}");
    return releaseSnapShots;
  }

  private ReleaseSnapshotDTO assembleReleaseSnapShot(long releaseId, String clusterName,
                                                     String configurations) {
    ReleaseSnapshotDTO releaseSnapShot = new ReleaseSnapshotDTO();
    releaseSnapShot.setReleaseId(releaseId);
    releaseSnapShot.setClusterName(clusterName);
    releaseSnapShot.setConfigurations(configurations);
    return releaseSnapShot;
  }

  private ClusterDTO[] assembleClusters() {
    ClusterDTO[] clusters = new ClusterDTO[2];
    clusters[0] = assembleCluster(100, 6666, Constants.DEFAULT_CLUSTER_NAME);
    clusters[1] = assembleCluster(101, 6666, "cluster1");
    return clusters;
  }

  private ClusterDTO assembleCluster(long id, long appId, String name) {
    ClusterDTO cluster = new ClusterDTO();
    cluster.setAppId(appId);
    cluster.setId(id);
    cluster.setName(name);
    return cluster;
  }

  private ConfigItemDTO[] assembleConfigItems() {
    ConfigItemDTO[] configItems = new ConfigItemDTO[5];
    configItems[0] =
        assembleConfigItem(100, Constants.DEFAULT_CLUSTER_NAME, 6666, "6666.k1", "6666.v1");
    configItems[1] =
        assembleConfigItem(100, Constants.DEFAULT_CLUSTER_NAME, 6666, "6666.k2", "6666.v2");
    configItems[2] =
        assembleConfigItem(100, Constants.DEFAULT_CLUSTER_NAME, 6666, "6666.k3", "6666.v3");
    configItems[3] =
        assembleConfigItem(100, Constants.DEFAULT_CLUSTER_NAME, 5555, "5555.k1", "5555.v1");
    configItems[4] = assembleConfigItem(101, "cluster1", 6666, "6666.k1", "6666.v1");
    return configItems;
  }

  private ConfigItemDTO assembleConfigItem(long clusterId, String clusterName, int appId,
                                           String key, String value) {
    ConfigItemDTO configItem = new ConfigItemDTO();
    configItem.setClusterName(clusterName);
    configItem.setClusterId(clusterId);
    configItem.setAppId(appId);
    configItem.setKey(key);
    configItem.setValue(value);
    return configItem;
  }

}
