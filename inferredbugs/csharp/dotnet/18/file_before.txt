using System;
using System.IO;
using System.Linq;
using Xunit;

namespace Structurizr.IO.PlantUML.Tests
{
    
    public class PlantUMLWriterTests
    {

        private PlantUMLWriter _plantUMLWriter;
        private Workspace _workspace;
        private StringWriter _stringWriter;

        public PlantUMLWriterTests()
        {
            _plantUMLWriter = new PlantUMLWriter();
            _workspace = new Workspace("Name", "Description");
            _stringWriter = new StringWriter();
        }

        [Fact]
        public void Test_WriteWorkspace_DoesNotThrowAnExceptionWhenPassedNullParameters()
        {
            _plantUMLWriter.Write((Workspace)null, null);
            _plantUMLWriter.Write(_workspace, null);
            _plantUMLWriter.Write((Workspace)null, _stringWriter);
        }

        [Fact]
        public void test_writeWorkspace()
        {
            PopulateWorkspace();
    
            _plantUMLWriter.Write(_workspace, _stringWriter);
            Assert.Equal("@startuml" + Environment.NewLine +
                    "title System Landscape for Some Enterprise" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "package \"Some Enterprise\" {" + Environment.NewLine +
                    "  actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "  component \"Software System\" <<Software System>> as 2" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "2 ..> 4 : Sends e-mail using" + Environment.NewLine +
                    "1 ..> 2 : Uses" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    "" + Environment.NewLine +
                    "@startuml" + Environment.NewLine +
                    "title Software System - System Context" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "component \"Software System\" <<Software System>> as 2" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "2 ..> 4 : Sends e-mail using" + Environment.NewLine +
                    "1 ..> 2 : Uses" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    "" + Environment.NewLine +
                    "@startuml" + Environment.NewLine +
                    "title Software System - Containers" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "package SoftwareSystem {" + Environment.NewLine +
                    "  component \"Database\" <<Container>> as 8" + Environment.NewLine +
                    "  component \"Web Application\" <<Container>> as 7" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "1 ..> 7 : Uses <<HTTP>>" + Environment.NewLine +
                    "7 ..> 8 : Reads from and writes to <<JDBC>>" + Environment.NewLine +
                    "7 ..> 4 : Sends e-mail using" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    "" + Environment.NewLine +
                    "@startuml" + Environment.NewLine +
                    "title Software System - Web Application - Components" + Environment.NewLine +
                    "component \"Database\" <<Container>> as 8" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "package WebApplication {" + Environment.NewLine +
                    "  component \"EmailComponent\" <<Component>> as 13" + Environment.NewLine +
                    "  component \"SomeController\" <<Spring MVC Controller>> as 12" + Environment.NewLine +
                    "  component \"SomeRepository\" <<Spring Data>> as 14" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "13 ..> 4 : Sends e-mails using <<SMTP>>" + Environment.NewLine +
                    "12 ..> 13 : Sends e-mail using" + Environment.NewLine +
                    "12 ..> 14 : Uses" + Environment.NewLine +
                    "14 ..> 8 : Reads from and writes to <<JDBC>>" + Environment.NewLine +
                    "1 ..> 12 : Uses <<HTTP>>" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    "" + Environment.NewLine +
                    "@startuml" + Environment.NewLine +
                    "title Web Application - Dynamic" + Environment.NewLine +
                    "component \"Database\" <<Container>> as 8" + Environment.NewLine +
                    "component \"SomeController\" <<Spring MVC Controller>> as 12" + Environment.NewLine +
                    "component \"SomeRepository\" <<Spring Data>> as 14" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "1 -> 12 : Requests /something" + Environment.NewLine +
                    "12 -> 14 : Uses" + Environment.NewLine +
                    "14 -> 8 : select * from something" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    "" + Environment.NewLine +
                    "@startuml" + Environment.NewLine +
                    "title Software System - Deployment" + Environment.NewLine +
                    "node \"Database Server\" <<Ubuntu 12.04 LTS>> as 23 {" + Environment.NewLine +
                    "  node \"MySQL\" <<MySQL 5.5.x>> as 24 {" + Environment.NewLine +
                    "    artifact \"Database\" <<Container>> as 25" + Environment.NewLine +
                    "  }" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "node \"Web Server\" <<Ubuntu 12.04 LTS>> as 20 {" + Environment.NewLine +
                    "  node \"Apache Tomcat\" <<Apache Tomcat 8.x>> as 21 {" + Environment.NewLine +
                    "    artifact \"Web Application\" <<Container>> as 22" + Environment.NewLine +
                    "  }" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "22 ..> 25 : Reads from and writes to <<JDBC>>" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    Environment.NewLine, _stringWriter.ToString());
        }
    
        [Fact]
        public void test_writeEnterpriseContextView()
        {
            PopulateWorkspace();
    
            SystemLandscapeView systemLandscapeView = _workspace.Views.SystemLandscapeViews.First();
            _plantUMLWriter.Write(systemLandscapeView, _stringWriter);
    
            Assert.Equal("@startuml" + Environment.NewLine +
                    "title System Landscape for Some Enterprise" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "package \"Some Enterprise\" {" + Environment.NewLine +
                    "  actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "  component \"Software System\" <<Software System>> as 2" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "2 ..> 4 : Sends e-mail using" + Environment.NewLine +
                    "1 ..> 2 : Uses" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    Environment.NewLine, _stringWriter.ToString());
    
        }
    
        [Fact]
        public void test_writeSystemContextView()
        {
            PopulateWorkspace();
    
            SystemContextView systemContextView = _workspace.Views.SystemContextViews.First();
            _plantUMLWriter.Write(systemContextView, _stringWriter);
    
            Assert.Equal("@startuml" + Environment.NewLine +
                    "title Software System - System Context" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "component \"Software System\" <<Software System>> as 2" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "2 ..> 4 : Sends e-mail using" + Environment.NewLine +
                    "1 ..> 2 : Uses" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    Environment.NewLine, _stringWriter.ToString());
        }
    
        [Fact]
        public void test_writeContainerView() 
        {
            PopulateWorkspace();
    
            ContainerView containerView = _workspace.Views.ContainerViews.First();
            _plantUMLWriter.Write(containerView, _stringWriter);
    
            Assert.Equal("@startuml" + Environment.NewLine +
                    "title Software System - Containers" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "package SoftwareSystem {" + Environment.NewLine +
                    "  component \"Database\" <<Container>> as 8" + Environment.NewLine +
                    "  component \"Web Application\" <<Container>> as 7" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "1 ..> 7 : Uses <<HTTP>>" + Environment.NewLine +
                    "7 ..> 8 : Reads from and writes to <<JDBC>>" + Environment.NewLine +
                    "7 ..> 4 : Sends e-mail using" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    Environment.NewLine, _stringWriter.ToString());
        }
    
        [Fact]
        public void test_writeComponentsView()
        {
            PopulateWorkspace();
    
            ComponentView componentView = _workspace.Views.ComponentViews.First();
            _plantUMLWriter.Write(componentView, _stringWriter);
    
            Assert.Equal("@startuml" + Environment.NewLine +
                    "title Software System - Web Application - Components" + Environment.NewLine +
                    "component \"Database\" <<Container>> as 8" + Environment.NewLine +
                    "component \"E-mail System\" <<Software System>> as 4" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "package WebApplication {" + Environment.NewLine +
                    "  component \"EmailComponent\" <<Component>> as 13" + Environment.NewLine +
                    "  component \"SomeController\" <<Spring MVC Controller>> as 12" + Environment.NewLine +
                    "  component \"SomeRepository\" <<Spring Data>> as 14" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "4 ..> 1 : Delivers e-mails to" + Environment.NewLine +
                    "13 ..> 4 : Sends e-mails using <<SMTP>>" + Environment.NewLine +
                    "12 ..> 13 : Sends e-mail using" + Environment.NewLine +
                    "12 ..> 14 : Uses" + Environment.NewLine +
                    "14 ..> 8 : Reads from and writes to <<JDBC>>" + Environment.NewLine +
                    "1 ..> 12 : Uses <<HTTP>>" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    Environment.NewLine, _stringWriter.ToString());
        }
    
        [Fact]
        public void test_writeDynamicView()
        {
            PopulateWorkspace();
    
            DynamicView dynamicView = _workspace.Views.DynamicViews.First();
            _plantUMLWriter.Write(dynamicView, _stringWriter);
    
            Assert.Equal("@startuml" + Environment.NewLine +
                    "title Web Application - Dynamic" + Environment.NewLine +
                    "component \"Database\" <<Container>> as 8" + Environment.NewLine +
                    "component \"SomeController\" <<Spring MVC Controller>> as 12" + Environment.NewLine +
                    "component \"SomeRepository\" <<Spring Data>> as 14" + Environment.NewLine +
                    "actor \"User\" <<Person>> as 1" + Environment.NewLine +
                    "1 -> 12 : Requests /something" + Environment.NewLine +
                    "12 -> 14 : Uses" + Environment.NewLine +
                    "14 -> 8 : select * from something" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    Environment.NewLine, _stringWriter.ToString());
        }
 
        [Fact]
        public void test_writeDeploymentView()
        {
            PopulateWorkspace();
    
            DeploymentView deploymentView = _workspace.Views.DeploymentViews.First();
            _plantUMLWriter.Write(deploymentView, _stringWriter);
    
            Assert.Equal("@startuml" + Environment.NewLine +
                    "title Software System - Deployment" + Environment.NewLine +
                    "node \"Database Server\" <<Ubuntu 12.04 LTS>> as 23 {" + Environment.NewLine +
                    "  node \"MySQL\" <<MySQL 5.5.x>> as 24 {" + Environment.NewLine +
                    "    artifact \"Database\" <<Container>> as 25" + Environment.NewLine +
                    "  }" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "node \"Web Server\" <<Ubuntu 12.04 LTS>> as 20 {" + Environment.NewLine +
                    "  node \"Apache Tomcat\" <<Apache Tomcat 8.x>> as 21 {" + Environment.NewLine +
                    "    artifact \"Web Application\" <<Container>> as 22" + Environment.NewLine +
                    "  }" + Environment.NewLine +
                    "}" + Environment.NewLine +
                    "22 ..> 25 : Reads from and writes to <<JDBC>>" + Environment.NewLine +
                    "@enduml" + Environment.NewLine +
                    Environment.NewLine, _stringWriter.ToString());
        }
    
        private void PopulateWorkspace()
        {
            Model model = _workspace.Model;
            ViewSet views = _workspace.Views;
            model.Enterprise = new Enterprise("Some Enterprise");
    
            Person user = model.AddPerson(Location.Internal, "User", "");
            SoftwareSystem softwareSystem = model.AddSoftwareSystem(Location.Internal, "Software System", "");
            user.Uses(softwareSystem, "Uses");
    
            SoftwareSystem emailSystem = model.AddSoftwareSystem(Location.External, "E-mail System", "");
            softwareSystem.Uses(emailSystem, "Sends e-mail using");
            emailSystem.Delivers(user, "Delivers e-mails to");
    
            Container webApplication = softwareSystem.AddContainer("Web Application", "", "");
            Container database = softwareSystem.AddContainer("Database", "", "");
            user.Uses(webApplication, "Uses", "HTTP");
            webApplication.Uses(database, "Reads from and writes to", "JDBC");
            webApplication.Uses(emailSystem, "Sends e-mail using");
    
            Component controller = webApplication.AddComponent("SomeController", "", "Spring MVC Controller");
            Component emailComponent = webApplication.AddComponent("EmailComponent", "");
            Component repository = webApplication.AddComponent("SomeRepository", "", "Spring Data");
            user.Uses(controller, "Uses", "HTTP");
            controller.Uses(repository, "Uses");
            controller.Uses(emailComponent, "Sends e-mail using");
            repository.Uses(database, "Reads from and writes to", "JDBC");
            emailComponent.Uses(emailSystem, "Sends e-mails using", "SMTP");

            DeploymentNode webServer = model.AddDeploymentNode("Web Server", "A server hosted at AWS EC2.", "Ubuntu 12.04 LTS");
            webServer.AddDeploymentNode("Apache Tomcat", "The live web server", "Apache Tomcat 8.x")
                    .Add(webApplication);
            DeploymentNode databaseServer = model.AddDeploymentNode("Database Server", "A server hosted at AWS EC2.", "Ubuntu 12.04 LTS");
            databaseServer.AddDeploymentNode("MySQL", "The live database server", "MySQL 5.5.x")
                    .Add(database);
    
            SystemLandscapeView
                systemLandscapeView = views.CreateSystemLandscapeView("enterpriseContext", "");
            systemLandscapeView.AddAllElements();
    
            SystemContextView systemContextView = views.CreateSystemContextView(softwareSystem, "systemContext", "");
            systemContextView.AddAllElements();
    
            ContainerView containerView = views.CreateContainerView(softwareSystem, "containers", "");
            containerView.AddAllElements();
    
            ComponentView componentView = views.CreateComponentView(webApplication, "components", "");
            componentView.AddAllElements();
    
            DynamicView dynamicView = views.CreateDynamicView(webApplication, "dynamic", "");
            dynamicView.Add(user, "Requests /something", controller);
            dynamicView.Add(controller, repository);
            dynamicView.Add(repository, "select * from something", database);

            DeploymentView deploymentView = views.CreateDeploymentView(softwareSystem, "deployment", "");
            deploymentView.AddAllDeploymentNodes();
        }

    }
}