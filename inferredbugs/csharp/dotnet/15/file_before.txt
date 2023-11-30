using Structurizr.Api;

namespace Structurizr.Examples
{
    
    /// <summary>
    /// A simple example of what a microservices architecture might look like. This workspace also
    /// includes a dynamic view that demonstrates parallel sequences of events.
    ///
    /// The live version of the diagrams can be found at https://structurizr.com/public/4241
    /// </summary>
    public class MicroservicesExample
    {
       
        private const long WorkspaceId = 4241;
        private const string ApiKey = "key";
        private const string ApiSecret = "secret";

        private const string MicroserviceTag = "Microservice";
        private const string MessageBusTag = "Message Bus";
        private const string DataStoreTag = "Database";

        static void Main()
        {
            Workspace workspace = new Workspace("Microservices example", "An example of a microservices architecture, which includes asynchronous and parallel behaviour.");
            Model model = workspace.Model;
    
            SoftwareSystem mySoftwareSystem = model.AddSoftwareSystem("Customer Information System", "Stores information ");
            Person customer = model.AddPerson("Customer", "A customer");
            Container customerApplication = mySoftwareSystem.AddContainer("Customer Application", "Allows customers to manage their profile.", "Angular");
    
            Container customerService = mySoftwareSystem.AddContainer("Customer Service", "The point of access for customer information.", "Java and Spring Boot");
            customerService.AddTags(MicroserviceTag);
            Container customerDatabase = mySoftwareSystem.AddContainer("Customer Database", "Stores customer information.", "Oracle 12c");
            customerDatabase.AddTags(DataStoreTag);
    
            Container reportingService = mySoftwareSystem.AddContainer("Reporting Service", "Creates normalised data for reporting purposes.", "Ruby");
            reportingService.AddTags(MicroserviceTag);
            Container reportingDatabase = mySoftwareSystem.AddContainer("Reporting Database", "Stores a normalised version of all business data for ad hoc reporting purposes.", "MySQL");
            reportingDatabase.AddTags(DataStoreTag);
    
            Container auditService = mySoftwareSystem.AddContainer("Audit Service", "Provides organisation-wide auditing facilities.", "C# .NET");
            auditService.AddTags(MicroserviceTag);
            Container auditStore = mySoftwareSystem.AddContainer("Audit Store", "Stores information about events that have happened.", "Event Store");
            auditStore.AddTags(DataStoreTag);
    
            Container messageBus = mySoftwareSystem.AddContainer("Message Bus", "Transport for business events.", "RabbitMQ");
            messageBus.AddTags(MessageBusTag);
    
            customer.Uses(customerApplication, "Uses");
            customerApplication.Uses(customerService, "Updates customer information using", "JSON/HTTPS", InteractionStyle.Synchronous);
            customerService.Uses(messageBus, "Sends customer update events to", "", InteractionStyle.Asynchronous);
            customerService.Uses(customerDatabase, "Stores data in", "JDBC", InteractionStyle.Synchronous);
            customerService.Uses(customerApplication, "Sends events to", "WebSocket", InteractionStyle.Asynchronous);
            messageBus.Uses(reportingService, "Sends customer update events to", "", InteractionStyle.Asynchronous);
            messageBus.Uses(auditService, "Sends customer update events to", "", InteractionStyle.Asynchronous);
            reportingService.Uses(reportingDatabase, "Stores data in", "", InteractionStyle.Synchronous);
            auditService.Uses(auditStore, "Stores events in", "", InteractionStyle.Synchronous);
    
            ViewSet views = workspace.Views;
    
            ContainerView containerView = views.CreateContainerView(mySoftwareSystem, "Containers", null);
            containerView.AddAllElements();
    
            DynamicView dynamicView = views.CreateDynamicView(mySoftwareSystem, "CustomerUpdateEvent", "This diagram shows what happens when a customer updates their details.");
            dynamicView.Add(customer, customerApplication);
            dynamicView.Add(customerApplication, customerService);
    
            dynamicView.Add(customerService, customerDatabase);
            dynamicView.Add(customerService, messageBus);
    
            dynamicView.StartParallelSequence();
            dynamicView.Add(messageBus, reportingService);
            dynamicView.Add(reportingService, reportingDatabase);
            dynamicView.EndParallelSequence();
    
            dynamicView.StartParallelSequence();
            dynamicView.Add(messageBus, auditService);
            dynamicView.Add(auditService, auditStore);
            dynamicView.EndParallelSequence();
    
            dynamicView.Add(customerService, "Confirms update to", customerApplication);

            Styles styles = views.Configuration.Styles;
            styles.Add(new ElementStyle(Tags.Element) { Color = "#000000" });
            styles.Add(new ElementStyle(Tags.Person) { Background = "#ffbf00", Shape = Shape.Person });
            styles.Add(new ElementStyle(Tags.Container) { Background = "#facc2E" });
            styles.Add(new ElementStyle(MessageBusTag) {Width = 1600, Shape = Shape.Pipe});
            styles.Add(new ElementStyle(MicroserviceTag) { Shape = Shape.Hexagon});
            styles.Add(new ElementStyle(DataStoreTag) { Background = "#f5da81", Shape = Shape.Cylinder });
            styles.Add(new RelationshipStyle(Tags.Relationship) { Routing = Routing.Orthogonal });
    
            styles.Add(new RelationshipStyle(Tags.Asynchronous) { Dashed = true });
            styles.Add(new RelationshipStyle(Tags.Synchronous) { Dashed = false });

            StructurizrClient structurizrClient = new StructurizrClient(ApiKey, ApiSecret);
            structurizrClient.PutWorkspace(WorkspaceId, workspace);
        }

    }
    
}