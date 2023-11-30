using IFramework;
using IFramework.MVVM;

namespace Example
{
    public class MvvmTest : Test
    {
        class MyModel : IModel
        {
            public int value=2;
        }
        class MyViewModel : ViewModel<MyModel>
        {
            private int _value;
            public int value { get { return GetProperty(ref _value); } set {

                    Tmodel.value = value;
                    SetProperty(ref _value, value); } }

            protected override void Initialize()
            {
                value = 5;
            }

            protected override void Listen(IEventArgs message)
            {
                Log.E("Message rec");
            }

            protected override void SyncModelValue()
            {
                value = Tmodel.value;
                this.value++;
            }

        }
        class MyView : View<MyViewModel>
        {
            protected override void BindProperty()
            {
                base.BindProperty();
                this.handler.BindProperty(() => { var value = Tcontext.value;

                    Log.E(value);
                });
                Log.E("Message pub");
                Publish(null);
            }
           
        }
        protected override void Start()
        {
            new MVVMGroup("name", new MyView(), new MyViewModel(), new MyModel());
        }

        protected override void Stop()
        {
        }

        protected override void Update()
        {
        }
    }



}
