using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Logging;
using Sir.Search;
using Sir.VectorSpace;
using System;
using System.IO;

namespace Sir.HttpServer
{
    public static class ServiceConfiguration
    {
        public static IServiceProvider Configure(IServiceCollection services)
        {
            var assemblyPath = Directory.GetCurrentDirectory();
            var config = new KeyValueConfiguration(Path.Combine(assemblyPath, "sir.ini"));

            services.Add(new ServiceDescriptor(typeof(IConfigurationProvider), config));

            var loggerFactory = services.BuildServiceProvider().GetService<ILoggerFactory>();
            var logger = loggerFactory.CreateLogger("Sir");
            var model = new BagOfCharsModel();
            var sessionFactory = new SessionFactory(@"c:\data\resin", logger);
            var qp = new QueryParser<string>(sessionFactory, model, logger);
            var httpParser = new HttpQueryParser(qp);

            services.AddSingleton(typeof(IModel<string>), model);
            services.AddSingleton(typeof(ISessionFactory), sessionFactory);
            services.AddSingleton(typeof(SessionFactory), sessionFactory);
            services.AddSingleton(typeof(QueryParser<string>), qp);
            services.AddSingleton(typeof(HttpQueryParser), httpParser);
            services.AddSingleton(typeof(IHttpWriter), new HttpWriter(sessionFactory));
            services.AddSingleton(typeof(IHttpReader), new HttpReader(
                sessionFactory,
                httpParser,
                loggerFactory.CreateLogger<HttpReader>()));

            return services.BuildServiceProvider();
        }
    }
}
