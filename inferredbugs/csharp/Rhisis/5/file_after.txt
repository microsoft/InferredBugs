using Microsoft.Extensions.DependencyInjection;
using Rhisis.Game.Abstractions.Entities;
using Rhisis.Game.Abstractions.Features;
using Rhisis.Game.Abstractions.Systems;
using Rhisis.Game.Common;
using Rhisis.Game.Features;
using Rhisis.Network;
using Rhisis.Network.Snapshots;
using System;

namespace Rhisis.Game.Abstractions.Components
{
    public class Health : GameFeature, IHealth
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

            SendPacketToVisible(_mover, healthSnapshot, sendToPlayer: true);
        }

        public void Die(IMover killer, ObjectMessageType objectMessageType = ObjectMessageType.OBJMSG_ATK1, bool sendHitPoints = false)
        {
            Hp = 0;

            if (_mover is IPlayer && killer is IPlayer)
            {
                // TODO: PVP
            }
            else
            {
                using var moverDeathSnapshot = new FFSnapshot();
                moverDeathSnapshot.Merge(new MoverDeathSnapshot(_mover, killer, objectMessageType));

                if (sendHitPoints)
                {
                    moverDeathSnapshot.Merge(new UpdateParamPointSnapshot(_mover, DefineAttributes.HP, Hp));
                }

                SendPacketToVisible(_mover, moverDeathSnapshot, sendToPlayer: true);
            }

            _mover.Behavior.OnKilled(killer);
            killer.Behavior.OnTargetKilled(_mover);
        }

        public void SufferDamages(IMover attacker, int damages, AttackFlags attackFlags = AttackFlags.AF_GENERIC, ObjectMessageType objectMessageType = ObjectMessageType.OBJMSG_ATK1)
        {
            int damagesToInflict = Math.Min(Hp, damages);

            using var damageSnapshots = new FFSnapshot();

            if (damagesToInflict > 0)
            {
                Hp -= damagesToInflict;
                damageSnapshots.Merge(new UpdateParamPointSnapshot(_mover, DefineAttributes.HP, Hp));
            }

            damageSnapshots.Merge(new AddDamageSnapshot(_mover, attacker, attackFlags, damagesToInflict));

            SendPacketToVisible(_mover, damageSnapshots, sendToPlayer: true);

            if (IsDead)
            {
                Die(attacker, objectMessageType, sendHitPoints: true);
            }
        }
    }
}
