using Microsoft.Extensions.DependencyInjection;
using Rhisis.Game.Abstractions.Entities;
using Rhisis.Game.Abstractions.Features;
using Rhisis.Game.Abstractions.Systems;
using Rhisis.Game.Common;
using Rhisis.Network;
using Rhisis.Network.Snapshots;
using System;

namespace Rhisis.Game.Abstractions.Components
{
    public class Health : IHealth
    {
        private readonly IMover _mover;
        private readonly Lazy<IHealthFormulas> _healthFormulas;

        public bool IsDead => Hp < 0;

        public int Hp { get; set; }

        public int Mp { get; set; }

        public int Fp { get; set; }

        public int MaxHp => _healthFormulas.Value.GetMaxHp(_mover);

        public int MaxMp => _healthFormulas.Value.GetMaxMp(_mover);

        public int MaxFp => _healthFormulas.Value.GetMaxFp(_mover);

        /// <summary>
        /// Creates a new <see cref="Health"/> instance.
        /// </summary>
        /// <param name="player">Current player.</param>
        public Health(IMover mover)
        {
            _mover = mover;
            _healthFormulas = new Lazy<IHealthFormulas>(() => mover.Systems.GetService<IHealthFormulas>());
        }

        public void RegenerateAll()
        {
            Hp = MaxHp;
            Mp = MaxMp;
            Fp = MaxFp;

            using var healthSnapshot = new FFSnapshot();
            healthSnapshot.Merge(new UpdateParamPointSnapshot(_mover, DefineAttributes.HP, Hp));
            healthSnapshot.Merge(new UpdateParamPointSnapshot(_mover, DefineAttributes.MP, Mp));
            healthSnapshot.Merge(new UpdateParamPointSnapshot(_mover, DefineAttributes.FP, Fp));

            _mover.SendToVisible(healthSnapshot);

            if (_mover is IPlayer)
            {
                _mover.Send(healthSnapshot);
            }

        }
    }
}
