using Microsoft.Extensions.DependencyInjection;

namespace Sir.Store
{
    /// <summary>
    /// Initialize app.
    /// </summary>
    public class Start : IPluginStart
    {
        public void OnApplicationStartup(
            IServiceCollection services, ServiceProvider serviceProvider, IConfigurationProvider config)
        {
            var model = new BocModel();
            var httpParser = new HttpQueryParser(new QueryParser());
            var sessionFactory = new SessionFactory(config, model);

            services.AddSingleton(typeof(IStringModel), model);
            services.AddSingleton(typeof(SessionFactory), sessionFactory);
            services.AddSingleton(typeof(HttpQueryParser), httpParser);
            services.AddSingleton(typeof(IQueryFormatter), new QueryFormatter());
            services.AddSingleton(typeof(IWriter), new StoreWriter(sessionFactory));
            services.AddSingleton(typeof(IReader), new StoreReader(sessionFactory, httpParser));
        }
    }
}