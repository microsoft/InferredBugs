using System;
using System.Linq;
using Xunit;

namespace Structurizr.Core.Tests
{
    public class ModelTests : AbstractTestBase
    {
        
        [Fact]
        public void Test_AddContainerInstance_ThrowsAnException_WhenANullContainerIsSpecified() {
            try {
                Model.AddContainerInstance(null);
                throw new TestFailedException();
            } catch (ArgumentException ae) {
                Assert.Equal("A container must be specified.", ae.Message);
            }
        }

        [Fact]
        public void Test_AddContainerInstance_AddsAContainerInstance_WhenAContainerIsSpecified() {
            SoftwareSystem softwareSystem1 = Model.AddSoftwareSystem("Software System 1", "Description");
            Container container1 = softwareSystem1.AddContainer("Container 1", "Description", "Technology");
    
            SoftwareSystem softwareSystem2 = Model.AddSoftwareSystem("Software System 2", "Description");
            Container container2 = softwareSystem2.AddContainer("Container 2", "Description", "Technology");
    
            SoftwareSystem softwareSystem3 = Model.AddSoftwareSystem("Software System 3", "Description");
            Container container3 = softwareSystem3.AddContainer("Container 3", "Description", "Technology");
    
            container1.Uses(container2, "Uses 1", "Technology 1", InteractionStyle.Synchronous);
            container2.Uses(container3, "Uses 2", "Technology 2", InteractionStyle.Asynchronous);
    
            ContainerInstance containerInstance1 = Model.AddContainerInstance(container1);
            ContainerInstance containerInstance2 = Model.AddContainerInstance(container2);
            ContainerInstance containerInstance3 = Model.AddContainerInstance(container3);
    
            Assert.Same(container2, containerInstance2.Container);
            Assert.Equal(container2.Id, containerInstance2.ContainerId);
            Assert.Same(softwareSystem2, containerInstance2.Parent);
            Assert.Equal("/Software System 2/Container 2[1]", containerInstance2.CanonicalName);
            Assert.Equal("Container Instance", containerInstance2.Tags);
    
            Assert.Equal(1, containerInstance1.Relationships.Count);
            Relationship relationship = containerInstance1.Relationships.First();
            Assert.Same(containerInstance1, relationship.Source);
            Assert.Same(containerInstance2, relationship.Destination);
            Assert.Equal("Uses 1", relationship.Description);
            Assert.Equal("Technology 1", relationship.Technology);
            Assert.Equal(InteractionStyle.Synchronous, relationship.InteractionStyle);
    
            Assert.Equal(1, containerInstance2.Relationships.Count);
            relationship = containerInstance2.Relationships.First();
            Assert.Same(containerInstance2, relationship.Source);
            Assert.Same(containerInstance3, relationship.Destination);
            Assert.Equal("Uses 2", relationship.Description);
            Assert.Equal("Technology 2", relationship.Technology);
            Assert.Equal(InteractionStyle.Asynchronous, relationship.InteractionStyle);
        }
    
        [Fact]
        public void Test_AddImplicitRelationships_SetsTheDescriptionOfThePropagatedRelationship_WhenThereIsOnlyOnePossibleDescription() {
            Person user = Model.AddPerson("Person", "Description");
            SoftwareSystem softwareSystem = Model.AddSoftwareSystem("Software System", "Description");
            Container webApplication = softwareSystem.AddContainer("Web Application", "Description", "Technology");
    
            user.Uses(webApplication, "Uses", "");
            Model.AddImplicitRelationships();
    
            Assert.Equal(2, user.Relationships.Count);
    
            Relationship relationship = user.Relationships.First(r => r.Destination == softwareSystem);
            Assert.Equal("Uses", relationship.Description);
        }

        [Fact]
        public void Test_AddImplicitRelationships_DoeNotSetTheDescriptionOfThePropagatedRelationship_WhenThereIsMoreThanOnePossibleDescription() {
            Person user = Model.AddPerson("Person", "Description");
            SoftwareSystem softwareSystem = Model.AddSoftwareSystem("Software System", "Description");
            Container webApplication = softwareSystem.AddContainer("Web Application", "Description", "Technology");
    
            user.Uses(webApplication, "Does something", "");
            user.Uses(webApplication, "Does something else", "");
            Model.AddImplicitRelationships();
    
            Assert.Equal(3, user.Relationships.Count);
    
            Relationship relationship = user.Relationships.First(r => r.Destination == softwareSystem);
            Assert.Equal("", relationship.Description);
        }

        [Fact]
        public void Test_AddImplicitRelationships_SetsTheTechnologyOfThePropagatedRelationship_WhenThereIsOnlyOnePossibleTechnology() {
            Person user = Model.AddPerson("Person", "Description");
            SoftwareSystem softwareSystem = Model.AddSoftwareSystem("Software System", "Description");
            Container webApplication = softwareSystem.AddContainer("Web Application", "Description", "Technology");
    
            user.Uses(webApplication, "Uses", "HTTPS");
            Model.AddImplicitRelationships();
    
            Assert.Equal(2, user.Relationships.Count);
    
            Relationship relationship = user.Relationships.First(r => r.Destination == softwareSystem);
            Assert.Equal("HTTPS", relationship.Technology);
        }

        [Fact]
        public void Test_AddImplicitRelationships_DoeNotSetTheTechnologyOfThePropagatedRelationship_WhenThereIsMoreThanOnePossibleTechnology() {
            Person user = Model.AddPerson("Person", "Description");
            SoftwareSystem softwareSystem = Model.AddSoftwareSystem("Software System", "Description");
            Container webApplication = softwareSystem.AddContainer("Web Application", "Description", "Technology");
    
            user.Uses(webApplication, "Does something", "Some technology");
            user.Uses(webApplication, "Does something else", "Some other technology");
            Model.AddImplicitRelationships();
    
            Assert.Equal(3, user.Relationships.Count);
    
            Relationship relationship = user.Relationships.First(r => r.Destination == softwareSystem);
            Assert.Equal("", relationship.Technology);
        }

        [Fact]
        public void Test_AddRelationship_DisallowsTheSameRelationshipToBeAddedMoreThanOnce()
        {
            SoftwareSystem element1 = Model.AddSoftwareSystem("Element 1", "Description");
            SoftwareSystem element2 = Model.AddSoftwareSystem("Element 2", "Description");
            Relationship relationship1 = element1.Uses(element2, "Uses", "");
            Relationship relationship2 = element1.Uses(element2, "Uses", "");
            Assert.True(element1.Has(relationship1));
            Assert.Null(relationship2);
            Assert.Equal(1, element1.Relationships.Count);
        }

        [Fact]
        public void Test_AddRelationship_AllowsMultipleRelationshipsBetweenElements()
        {
            SoftwareSystem element1 = Model.AddSoftwareSystem("Element 1", "Description");
            SoftwareSystem element2 = Model.AddSoftwareSystem("Element 2", "Description");
            Relationship relationship1 = element1.Uses(element2, "Uses in some way", "");
            Relationship relationship2 = element1.Uses(element2, "Uses in another way", "");
            Assert.True(element1.Has(relationship1));
            Assert.True(element1.Has(relationship2));
            Assert.Equal(2, element1.Relationships.Count);
        }

        [Fact]
        public void Test_ModifyRelationship_ThrowsAnException_WhenARelationshipIsNotSpecified()
        {
            try
            {
                Model.ModifyRelationship(null, "Uses", "Technology");
                throw new TestFailedException();
            }
            catch (ArgumentException ae)
            {
                Assert.Equal("A relationship must be specified.", ae.Message);
            }
        }

        [Fact]
        public void Test_ModifyRelationship_ModifiesAnExistingRelationship_WhenThatRelationshipDoesNotAlreadyExist()
        {
            SoftwareSystem element1 = Model.AddSoftwareSystem("Element 1", "Description");
            SoftwareSystem element2 = Model.AddSoftwareSystem("Element 2", "Description");
            Relationship relationship = element1.Uses(element2, "", "");

            Model.ModifyRelationship(relationship, "Uses", "Technology");
            Assert.Equal("Uses", relationship.Description);
            Assert.Equal("Technology", relationship.Technology);
        }

        [Fact]
        public void Test_ModifyRelationship_ThrowsAnException_WhenThatRelationshipDoesAlreadyExist()
        {
            SoftwareSystem element1 = Model.AddSoftwareSystem("Element 1", "Description");
            SoftwareSystem element2 = Model.AddSoftwareSystem("Element 2", "Description");
            Relationship relationship = element1.Uses(element2, "Uses", "Technology");

            try
            {
                Model.ModifyRelationship(relationship, "Uses", "Technology");
                throw new TestFailedException();
            }
            catch (ArgumentException ae)
            {
                Assert.Equal("This relationship exists already: {1 | Element 1 | Description} ---[Uses]---> {2 | Element 2 | Description}", ae.Message);
            }
        }



    }
}