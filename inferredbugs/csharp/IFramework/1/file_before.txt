using IFramework;

namespace Example
{
    public class MouduleTest : Test
    {
        private class MyModule : IFramework.Modules.UpdateModule
        {

            protected override void Awake()
            {
                Log.L("Awake ");
            }
            protected override void OnEnable()
            {
                Log.L("OnEnable ");

            }
            protected override void OnUpdate()
            {
                Log.L("OnUpdate ");

            }
            protected override void OnDisable()
            {
                Log.L("OnDisable ");

            }
            public void Say()
            {
                Log.L("Say ");

            }
            protected override void OnDispose()
            {
                Log.L("OnDispose ");

            }
        }
        protected override void Start()
        {
            Log.L("Create An module ");

            Framework.GetEnv(EnvironmentType.Ev0).modules.CreateModule<MyModule>();
            var module= Framework.GetEnv(EnvironmentType.Ev0).modules.FindModule<MyModule>();
            Log.L("the module id "+module);
            Log.L("say ");

            module.Say();
            Log.L("Dispose ");
            module.UnBind();
            Log.L("say ");
            module.Say();

            Log.L("Get An other ");
            for (int i = 0; i < 10; i++)
            {
               Log.L(Framework.GetEnv(EnvironmentType.Ev0).modules.GetModule<MyModule>("XXL"));
            }
        }

        protected override void Stop()
        {
        }

        protected override void Update()
        {
        }
    }
}
