using Microsoft.Extensions.DependencyInjection;
using System.IO;

namespace Sir.Store
{
    public class Start : IPluginStart
    {
        public void OnApplicationStartup(IServiceCollection services)
        {
            var tokenizer = new LatinTokenizer();

            services.AddSingleton(typeof(LocalStorageSessionFactory), new LocalStorageSessionFactory(Path.Combine(Directory.GetCurrentDirectory(), "App_Data"), tokenizer));

            services.AddSingleton(typeof(ITokenizer), tokenizer);

            services.AddSingleton(typeof(HttpQueryParser), new HttpQueryParser(new BooleanKeyValueQueryParser()));
        }
    }
}
