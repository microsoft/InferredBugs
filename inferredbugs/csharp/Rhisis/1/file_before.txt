using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;
using Rhisis.Core.Common;
using Rhisis.Core.DependencyInjection;
using Rhisis.Core.Helpers;
using Rhisis.Database.Context;
using System;

namespace Rhisis.Database
{
    public sealed class DatabaseFactory : Singleton<DatabaseFactory>, IDesignTimeDbContextFactory<DatabaseContext>
    {
        private const string MigrationConfigurationEnv = "DB_CONFIG";

        public DatabaseConfiguration Configuration { get; private set; }

        /// <summary>
        /// Initialize the <see cref="DatabaseFactory"/> and Data Access Layer assembly.
        /// </summary>
        public void Initialize()
        {
            DependencyContainer.Instance.Register<IDatabase, Database>();
        }

        /// <summary>
        /// Initialize the <see cref="DatabaseFactory"/>.
        /// </summary>
        /// <param name="databaseConfigurationPath"></param>
        public void Initialize(string databaseConfigurationPath)
        {
            this.Configuration = ConfigurationHelper.Load<DatabaseConfiguration>(databaseConfigurationPath);

            using (var dbContext = this.CreateDbContext())
            {
                dbContext.Migrate();
                if (!dbContext.DatabaseExists())
                    throw new InvalidOperationException($"The database '{this.Configuration.Database}' doesn't exists.");
            }

            this.Initialize();
        }

        /// <summary>
        /// Creates a new <see cref="DatabaseContext"/>.
        /// </summary>
        /// <returns></returns>
        public DatabaseContext CreateDbContext() => this.CreateDbContext(this.Configuration);

        /// <summary>
        /// Creates a new <see cref="DatabaseContext"/> with a specific configuration.
        /// </summary>
        /// <param name="configuration"></param>
        /// <returns></returns>
        public DatabaseContext CreateDbContext(DatabaseConfiguration configuration) => new DatabaseContext(configuration);

        /// <summary>
        /// Creates a new <see cref="DatabaseContext"/> by passing context options.
        /// </summary>
        /// <param name="options"></param>
        /// <returns></returns>
        public DatabaseContext CreateDbContext(DbContextOptions options) => new DatabaseContext(options);

        /// <inheritdoc />
        public DatabaseContext CreateDbContext(string[] args)
        {
            var configurationPath = Environment.GetEnvironmentVariable(MigrationConfigurationEnv);
            var configuration = ConfigurationHelper.Load<DatabaseConfiguration>(configurationPath);
            var context = new DatabaseContext(configuration);

            if (!string.IsNullOrEmpty(configurationPath) && !context.DatabaseExists())
                throw new InvalidOperationException($"The database '{configuration.Database}' doesn't exists.");

            return context;
        }
    }
}
