package cc.blynk.server.api.http.logic.business;

import cc.blynk.core.http.MediaType;
import cc.blynk.core.http.Response;
import cc.blynk.core.http.annotation.Consumes;
import cc.blynk.core.http.annotation.Context;
import cc.blynk.core.http.annotation.DELETE;
import cc.blynk.core.http.annotation.GET;
import cc.blynk.core.http.annotation.POST;
import cc.blynk.core.http.annotation.PUT;
import cc.blynk.core.http.annotation.Path;
import cc.blynk.core.http.annotation.PathParam;
import cc.blynk.core.http.annotation.QueryParam;
import cc.blynk.core.http.model.Filter;
import cc.blynk.server.core.dao.FileManager;
import cc.blynk.server.core.dao.SessionDao;
import cc.blynk.server.core.dao.TokenManager;
import cc.blynk.server.core.model.DashBoard;
import cc.blynk.server.core.model.auth.User;
import cc.blynk.utils.ArrayUtil;
import cc.blynk.utils.JsonParser;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.Arrays;
import java.util.List;

import static cc.blynk.core.http.Response.appendTotalCountHeader;
import static cc.blynk.core.http.Response.ok;
import static cc.blynk.utils.HttpLogicUtil.sort;

/**
 * The Blynk Project.
 * Created by Dmitriy Dumanskiy.
 * Created on 09.12.15.
 */
@Path("/projects")
public class BusinessLogic {

    private static final Logger log = LogManager.getLogger(BusinessLogic.class);

    private final TokenManager tokenManager;
    private final SessionDao sessionDao;
    private final FileManager fileManager;

    public BusinessLogic(TokenManager tokenManager, SessionDao sessionDao, FileManager fileManager) {
        this.tokenManager = tokenManager;
        this.fileManager = fileManager;
        this.sessionDao = sessionDao;
    }

    @GET
    @Path("")
    public Response getProjects(@Context User user,
                             @QueryParam("_filters") String filterParam,
                             @QueryParam("_page") int page,
                             @QueryParam("_perPage") int size,
                             @QueryParam("_sortField") String sortField,
                             @QueryParam("_sortDir") String sortOrder) {

        if (filterParam != null) {
            Filter filter = JsonParser.readAny(filterParam, Filter.class);
            filterParam = filter == null ? null : filter.name;
        }

        List<DashBoard> projects = Arrays.asList(user.profile.dashBoards);

        for (DashBoard project : projects) {
            project.token = user.dashTokens.get(project.id);
        }

        return appendTotalCountHeader(
                ok(sort(projects , sortField, sortOrder), page, size), projects.size()
        );
    }

    @GET
    @Path("/{projectId}")
    public Response getProject(@Context User user,
                             @PathParam("projectId") int projectId) {

        DashBoard project = user.profile.getDashById(projectId);
        project.token = user.dashTokens.get(projectId);

        return ok(project);
    }

    @POST
    @Consumes(value = MediaType.APPLICATION_JSON)
    @Path("")
    public Response createProject(@Context User user, DashBoard newProject) {

        log.debug("Creating project {}", newProject);

        newProject.createdAt = System.currentTimeMillis();
        newProject.updatedAt = newProject.createdAt;
        newProject.id = findMaxId(user.profile.dashBoards) + 1;
        String token = user.dashTokens.get(newProject.id);
        if (token == null) {
            tokenManager.refreshToken(user, newProject.id, 0);
        }

        user.profile.dashBoards = ArrayUtil.add(user.profile.dashBoards, newProject, DashBoard.class);
        user.lastModifiedTs = System.currentTimeMillis();

        return ok(newProject);
    }

    private static int findMaxId(DashBoard[] dashBoards) {
        int max = 0;
        for (DashBoard dashBoard : dashBoards) {
            max = Math.max(dashBoard.id, max);
        }
        return max;
    }

    @DELETE
    @Path("/{projectId}")
    public Response deleteProject(@Context User user, @PathParam("projectId") int projectId) {

        log.debug("Deleting project {}", projectId);

        int index = user.profile.getDashIndexOrThrow(projectId);
        DashBoard dash = user.profile.dashBoards[index];
        user.profile.dashBoards = ArrayUtil.remove(user.profile.dashBoards, index, DashBoard.class);

        tokenManager.deleteDash(user, dash);

        user.lastModifiedTs = System.currentTimeMillis();

        return ok();
    }

    @PUT
    @Consumes(value = MediaType.APPLICATION_JSON)
    @Path("/{projectId}")
    public Response updateProject(@Context User user,
                               @PathParam("projectId") int projectId,
                               DashBoard updatedProject) {

        log.debug("Updating project {}", projectId);

        int projectIndex = user.profile.getDashIndexOrThrow(projectId);
        user.profile.dashBoards[projectIndex] = updatedProject;
        user.profile.dashBoards[projectIndex].updatedAt = System.currentTimeMillis();

        return ok(updatedProject);
    }


}
