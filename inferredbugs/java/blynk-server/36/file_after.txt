package cc.blynk.server.workers;

import cc.blynk.server.core.dao.ReportingDao;
import cc.blynk.server.core.dao.UserDao;
import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.DataStream;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.server.core.model.widgets.Target;
import cc.blynk.server.core.model.widgets.Widget;
import cc.blynk.server.core.model.widgets.outputs.HistoryGraph;
import cc.blynk.server.core.model.widgets.outputs.graph.EnhancedHistoryGraph;
import cc.blynk.server.core.model.widgets.outputs.graph.GraphDataStream;
import cc.blynk.server.core.model.widgets.outputs.graph.GraphGranularityType;
import cc.blynk.server.core.model.widgets.ui.tiles.DeviceTiles;
import cc.blynk.server.core.model.widgets.ui.tiles.TileTemplate;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.HashSet;
import java.util.Set;

/**
 * Daily job used to clean reporting data that is not used by the history graphs
 * but stored anyway on the disk.
 *
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 04.01.18.
 */
public class HistoryGraphUnusedPinDataCleanerWorker implements Runnable {

    private static final Logger log = LogManager.getLogger(HistoryGraphUnusedPinDataCleanerWorker.class);

    private final UserDao userDao;
    private final ReportingDao reportingDao;

    private long lastStart;

    public HistoryGraphUnusedPinDataCleanerWorker(UserDao userDao, ReportingDao reportingDao) {
        this.userDao = userDao;
        this.reportingDao = reportingDao;
        this.lastStart = System.currentTimeMillis();

    }

    @Override
    public void run() {
        try {
            log.info("Start removing unused reporting data...");

            long now = System.currentTimeMillis();
            //todo
            //actually, it is better to do not save data for such pins
            //but fow now this approach is simpler and quicker
            int result = removeUnsedInHistoryGraphData();

            lastStart = now;

            log.info("Removed {} files. Time : {} ms.", result, System.currentTimeMillis() - now);
        } catch (Throwable t) {
            log.error("Error removing unused reporting data.", t);
        }
    }

    private int removeUnsedInHistoryGraphData() {
        int removedFilesCounter = 0;
        Set<String> doNotRemovePaths = new HashSet<>();

        for (User user : userDao.getUsers().values()) {
            //we don't want to do a lot of work here,
            //so we check only active profiles that actually write data
            if (user.isUpdated(lastStart)) {
                doNotRemovePaths.clear();
                try {
                    for (DashBoard dashBoard : user.profile.dashBoards) {
                        for (Widget widget : dashBoard.widgets) {
                            if (widget instanceof DeviceTiles) {
                                DeviceTiles deviceTiles = (DeviceTiles) widget;
                                for (TileTemplate tileTemplate : deviceTiles.templates) {
                                    for (Widget tilesWidget : tileTemplate.widgets) {
                                        add(doNotRemovePaths, dashBoard, tilesWidget);
                                    }
                                }
                            } else {
                                add(doNotRemovePaths, dashBoard, widget);
                            }
                        }
                    }

                    removedFilesCounter += reportingDao.delete(user,
                            reportingFile -> !doNotRemovePaths.contains(reportingFile.getFileName().toString()));
                } catch (Exception e) {
                    log.error("Error cleaning reporting record for user {}. {}", user.email, e.getMessage());
                }
            }
        }
        return removedFilesCounter;
    }

    private static void add(Set<String> doNotRemovePaths, DashBoard dash, Widget widget) {
        if (widget instanceof HistoryGraph) {
            HistoryGraph historyGraph = (HistoryGraph) widget;
            add(doNotRemovePaths, dash.id, historyGraph);
        } else if (widget instanceof EnhancedHistoryGraph) {
            EnhancedHistoryGraph enhancedHistoryGraph = (EnhancedHistoryGraph) widget;
            add(doNotRemovePaths, dash, enhancedHistoryGraph);
        }
    }

    private static void add(Set<String> doNotRemovePaths, DashBoard dash, EnhancedHistoryGraph graph) {
        for (GraphDataStream graphDataStream : graph.dataStreams) {
            if (graphDataStream != null && graphDataStream.dataStream != null && graphDataStream.dataStream.isValid()) {
                DataStream dataStream = graphDataStream.dataStream;
                Target target = dash.getTarget(graphDataStream.targetId);
                if (target != null) {
                    for (int deviceId : target.getAssignedDeviceIds()) {
                        for (GraphGranularityType type : GraphGranularityType.values()) {
                            String filename = ReportingDao.generateFilename(dash.id,
                                    deviceId,
                                    dataStream.pinType.pintTypeChar, dataStream.pin, type.label);
                            doNotRemovePaths.add(filename);
                        }
                    }
                }
            }
        }
    }

    //todo history graph is only for back compatibility and should be removed in future
    private static void add(Set<String> doNotRemovePaths, int dashId, HistoryGraph graph) {
        for (DataStream dataStream : graph.dataStreams) {
            if (dataStream.isValid()) {
                for (GraphGranularityType type : GraphGranularityType.values()) {
                    String filename = ReportingDao.generateFilename(dashId, graph.deviceId,
                            dataStream.pinType.pintTypeChar, dataStream.pin, type.label);
                    doNotRemovePaths.add(filename);
                }
            }
        }
    }

}
