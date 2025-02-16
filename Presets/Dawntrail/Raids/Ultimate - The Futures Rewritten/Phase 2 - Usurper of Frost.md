## [Beta] [Script] Diamond Dust
It highlights positions.
Configuration:
- Set all spread positions based on the 1st Icicle Impact locations.
```
https://github.com/PunishXIV/Splatoon/raw/main/SplatoonScripts/Duties/Dawntrail/The%20Futures%20Rewritten/P2%20Diamond%20Dust.cs
```

## [Beta] [Script] Mirror, Mirror
It highlights your next mirror positions.
Configuration:
- Set your first blue mirror.
```
https://github.com/PunishXIV/Splatoon/raw/main/SplatoonScripts/Duties/Dawntrail/The%20Futures%20Rewritten/P2%20Mirror%20Mirror.cs
```

## [JP] [Beta] [Script] Light Rampant JP
It highlights your tower.
Configuration:
- Set player names and directions.
```
https://github.com/PunishXIV/Splatoon/raw/main/SplatoonScripts/Duties/Dawntrail/The%20Futures%20Rewritten/P2%20Light%20Rampant%20JP.cs
```

## [Test] Light Rampant NAUR
```

using Dalamud.Game.ClientState.Objects.SubKinds;
using ECommons;
using ECommons.Configuration;
using ECommons.ExcelServices;
using ECommons.GameFunctions;
using ECommons.GameHelpers;
using ECommons.Hooks.ActionEffectTypes;
using ECommons.ImGuiMethods;
using ECommons.Logging;
using ECommons.MathHelpers;
using ImGuiNET;
using Splatoon;
using Splatoon.SplatoonScripting;
using Splatoon.SplatoonScripting.Priority;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Numerics;

namespace SplatoonScriptsOfficial.Duties.Dawntrail.The_Futures_Rewritten;
internal class P2_Light_Rampant :SplatoonScript
{
    #region enums
    private enum State
    {
        None = 0,
        LightRampantCasting,
        TetherSpawned,
        TowerSpawned,
        AoeSET1,
        AoeSET2,
        AoeSET3,
        AoeSET4,
        AoeSET5,
        WaitStack,
        AvoidAOE,
        TowerIn,
        Banish,
    }
    #endregion

    #region class
    public class Config :IEzConfig
    {
        public bool NorthSwap = false;
        public PriorityData Priority = new();
    }

    private class PartyData
    {
        public int index = 0;
        public bool Mine => this.EntityId == Player.Object.EntityId;
        public uint EntityId = 0;
        public IPlayerCharacter? Object => (IPlayerCharacter)this.EntityId.GetObject() ?? null;
        public uint TetherPairId1 = 0;
        public uint TetherPairId2 = 0;
        public Direction TowerDirection = Direction.None;

        public PartyData(uint entityId)
        {
            EntityId = entityId;
        }
    }
    #endregion

    #region const
    private readonly List<(Direction, Vector3)> TowerPos = new()
    {
        new (Direction.NorthWest, new Vector3(86, 0, 92)),
        new (Direction.North, new Vector3(100, 0, 100)),
        new (Direction.NorthEast, new Vector3(114, 0, 92)),
        new (Direction.SouthEast, new Vector3(114, 0, 108)),
        new (Direction.South, new Vector3(100, 0, 100)),
        new (Direction.SouthWest, new Vector3(86, 0, 108)),
    };
    #endregion

    #region public properties
    public override HashSet<uint>? ValidTerritories => [1238];
    public override Metadata? Metadata => new(1, "redmoon");
    #endregion

    #region private properties
    private State _state = State.None;
    private List<PartyData> _partyDataList = new();
    private int _onTetherCount = 0;
    private Config C => Controller.GetConfig<Config>();
    private string NewPlayer = "";
    #endregion

    #region public methods
    public override void OnSetup()
    {
        Controller.RegisterElement("Bait", new Element(0) { tether = true, radius = 3f, thicc = 6f });
        Controller.RegisterElement("BaitObject", new Element(1) { tether = true, refActorComparisonType = 2, radius = 0.5f, thicc = 6f });
        for (var i = 0; i < 8; i++)
        {
            Controller.RegisterElement($"Circle{i}", new Element(1) { radius = 5.0f, refActorComparisonType = 2, thicc = 2f, fillIntensity = 0.2f });
        }
        Controller.RegisterElement("DropSpotN1", new Element(0) { radius = 0.3f, thicc = 2f, Filled = true, fillIntensity = 1f });
        Controller.RegisterElement("DropSpotS1", new Element(0) { radius = 0.3f, thicc = 2f, Filled = true, fillIntensity = 1f });
    }

    public override void OnStartingCast(uint source, uint castId)
    {
        if (castId == 40212)
        {
            SetListEntityIdByJob();

            _state = State.LightRampantCasting;
        }
    }

    public override void OnMapEffect(uint position, ushort data1, ushort data2)
    {
        if (_state != State.TetherSpawned) return;
        if (position == 9 && data1 == 1 && data2 == 2)
        {
            var pc = GetMinedata();

            if (pc.TowerDirection != Direction.None)
            {
                Controller.GetElementByName("Bait").SetRefPosition(TowerPos.Find(x => x.Item1 == pc.TowerDirection).Item2);
                Controller.GetElementByName("Bait").tether = true;
                Controller.GetElementByName("Bait").Enabled = true;
            }
        }
    }

    public override void OnActionEffectEvent(ActionEffectSet set)
    {
        if (set.Action == null) return;
        var castId = set.Action.Value.RowId;

        if (castId == 40213)
        {
            this.OnReset();
        }
    }

    public override void OnUpdate()
    {
        Controller.GetRegisteredElements().Where(x => x.Value.Enabled).Each(x => x.Value.color = GradientColor.Get(0xFF00FF00.ToVector4(), 0xFF0000FF.ToVector4()).ToUint());
    }

    public override void OnReset()
    {
        _state = State.None;
        _onTetherCount = 0;
        _partyDataList.Clear();
        HideAllElements();
    }

    public override void OnTetherCreate(uint source, uint target, uint data2, uint data3, uint data5)
    {
        if (_state != State.LightRampantCasting) return;
        if (data2 == 0 && data3 == 110 && data5 == 15)
        {
            var partyData = _partyDataList.Find(x => x.EntityId == source);
            if (partyData == null) return;
            partyData.TetherPairId1 = target;
            _onTetherCount++;

            if (_onTetherCount == 6)
            {
                if (C.Priority.GetFirstValidList() == null)
                {
                    PluginLog.Error("PriorityList is null");
                    this.OnReset();
                }

                if (ParseTether())
                {
                    _state = State.TetherSpawned;
                }
                else
                {
                    _state = State.None;
                }
            }
        }
    }

    public override void OnSettingsDraw()
    {
        ImGui.Text("If NorthWest, NorthEast Swap, Check This box");
        ImGui.Checkbox("NorthSwap", ref C.NorthSwap);

        ImGui.Text("Set NorthWest ClockWise (ex. NorthWest -> North -> NorthEast -> SouthEast -> South -> SouthWest)");
        ImGui.Text("Swaper is NorthEast and SouthEast Players");
        C.Priority.Draw();
        if (ImGuiEx.CollapsingHeader("Debug"))
        {
            ImGui.Text($"State: {_state}");
            ImGui.Text($"OnTetherCount: {_onTetherCount}");
            ImGui.Text($"PartyDataList.Count: {_partyDataList.Count}");
            if (_partyDataList.Count <= 0) return;
            foreach (var pc in _partyDataList)
            {
                if (pc.EntityId == 0) continue;
                ImGui.Text($"PartyData: {pc.EntityId.GetObject().Name.ToString()}");
                if (pc.TetherPairId1 == 0) continue;
                ImGui.Text($"TetherPairId1: {pc.TetherPairId1.GetObject().Name.ToString()}");
                if (pc.TetherPairId2 == 0) continue;
                ImGui.Text($"TetherPairId2: {pc.TetherPairId2.GetObject().Name.ToString()}");
                ImGui.Text($"TowerDirection: {pc.TowerDirection}");
            }
        }
    }
    #endregion

    #region private methods
    private readonly Job[] THJobs = new Job[] { Job.WAR, Job.DRK, Job.GNB, Job.PLD, Job.WHM, Job.AST, Job.SCH, Job.SGE };
    private readonly Job[] DPSJobs = new Job[] { Job.SAM, Job.MNK, Job.DRG, Job.RPR, Job.NIN, Job.VPR, Job.BRD, Job.MCH, Job.DNC, Job.RDM, Job.SMN, Job.BLU, Job.PCT };
    private bool ParseTether()
    {
        foreach (var pc in _partyDataList)
        {
            if (pc.TetherPairId1 == 0) continue;

            var pair2 = _partyDataList.Find(x => x.TetherPairId1 == pc.EntityId);
            if (pair2 == null) continue;
            pc.TetherPairId2 = pair2.EntityId;
        }

        if (_partyDataList.Where(x => x.TetherPairId1 != 0 && x.TetherPairId2 != 0).Count() != 6) return false;

        var tetherList = _partyDataList.Where(x => x.TetherPairId1 != 0 && x.TetherPairId2 != 0).ToList();
        List<PartyData> tethersSortedList = new List<PartyData>();
        var priority = C.Priority.GetPlayers(_ => true);
        if (priority == null || priority.Count != 8)
        {
            PluginLog.Information("PriorityList is null");
            return false;
        }

        // tetherListをpriority順に並べる
        foreach (var pc in priority)
        {
            var tether = tetherList.Find(x => x.EntityId == pc.IGameObject.EntityId);
            if (tether != null)
            {
                tethersSortedList.Add(tether);
            }
        }

        var Index = 0;
        foreach (var pc in tethersSortedList)
        {
            if (pc == null)
            {
                PluginLog.Information($"EntityId: {pc.EntityId} {Index}");
            }
            Index++;
        }

        if (priority == null || priority.Count == 0) return false;

        // 0は北西
        var x = _partyDataList.Find(x => x.EntityId == tethersSortedList[0].EntityId.GetObject().EntityId).TowerDirection = (C.NorthSwap != true) ? Direction.NorthWest : Direction.NorthEast;

        // 1は南確定
        x = _partyDataList.Find(x => x.EntityId == tethersSortedList[1].EntityId.GetObject().EntityId).TowerDirection = Direction.South;

        // 2は北東
        x = _partyDataList.Find(x => x.EntityId == tethersSortedList[2].EntityId.GetObject().EntityId).TowerDirection = (C.NorthSwap != true) ? Direction.NorthEast : Direction.NorthWest;

        // 3は南東
        x = _partyDataList.Find(x => x.EntityId == tethersSortedList[3].EntityId.GetObject().EntityId).TowerDirection = (C.NorthSwap != true) ? Direction.SouthWest : Direction.SouthEast;

        // 4は北確定
        x = _partyDataList.Find(x => x.EntityId == tethersSortedList[4].EntityId.GetObject().EntityId).TowerDirection = Direction.North;

        // 5は南西
        x = _partyDataList.Find(x => x.EntityId == tethersSortedList[5].EntityId.GetObject().EntityId).TowerDirection = (C.NorthSwap != true) ? Direction.SouthEast : Direction.SouthWest;


        if (_partyDataList.Where(x => x.TowerDirection != Direction.None).Count() != 6) return false;
        else return true;
    }

    #endregion

    #region API
    public enum Direction
    {
        None = 0,
        North,
        NorthEast,
        East,
        SouthEast,
        South,
        SouthWest,
        West,
        NorthWest
    }

    public enum LR
    {
        Left = 0,
        Right
    }

    public class DirectionalVector
    {
        public Direction Direction { get; }
        public Vector3 Position { get; }

        public DirectionalVector(Direction direction, Vector3 position)
        {
            Direction = direction;
            Position = position;
        }

        public override string ToString()
        {
            return $"{Direction}: {Position}";
        }
    }

    private void SetListEntityIdByJob()
    {
        _partyDataList.Clear();

        // リストに８人分の初期インスタンス生成
        for (var i = 0; i < 8; i++)
        {
            _partyDataList.Add(new PartyData(0));
            _partyDataList[i].index = i;
        }

        foreach (var pc in FakeParty.Get())
        {
            var job = pc.GetJob();
            switch (job)
            {
                case Job.WAR:
                case Job.DRK:
                case Job.GNB:
                _partyDataList[0].EntityId = pc.EntityId;
                break;

                case Job.PLD:
                _partyDataList[1].EntityId = pc.EntityId;
                break;

                case Job.SAM:
                case Job.MNK:
                case Job.DRG:
                case Job.RPR:
                _partyDataList[2].EntityId = pc.EntityId;
                break;

                case Job.NIN:
                case Job.VPR:
                case Job.RDM:
                case Job.BLM:
                case Job.SMN:
                _partyDataList[3].EntityId = pc.EntityId;
                break;

                case Job.BRD:
                case Job.MCH:
                case Job.DNC:
                _partyDataList[4].EntityId = pc.EntityId;
                break;

                case Job.PCT:
                _partyDataList[5].EntityId = pc.EntityId;
                break;

                case Job.WHM:
                case Job.AST:
                _partyDataList[6].EntityId = pc.EntityId;
                break;

                case Job.SCH:
                case Job.SGE:
                _partyDataList[7].EntityId = pc.EntityId;
                break;
            }
        }
    }

    private PartyData GetMinedata() => _partyDataList.Find(x => x.Mine);

    private Direction DividePoint(Vector3 Position, float Distance, Vector3? Center = null)
    {
        // Distance, Centerの値を用いて、８方向のベクトルを生成
        var directionalVectors = GenerateDirectionalVectors(Distance, Center ?? new Vector3(100, 0, 100));

        // ８方向の内、最も近い方向ベクトルを取得
        var closestDirection = Direction.North;
        var closestDistance = float.MaxValue;
        foreach (var directionalVector in directionalVectors)
        {
            var distance = Vector3.Distance(Position, directionalVector.Position);
            if (distance < closestDistance)
            {
                closestDistance = distance;
                closestDirection = directionalVector.Direction;
            }
        }

        return closestDirection;
    }

    public static List<DirectionalVector> GenerateDirectionalVectors(float distance, Vector3? center = null)
    {
        var directionalVectors = new List<DirectionalVector>();

        // 各方向のオフセット計算
        foreach (Direction direction in Enum.GetValues(typeof(Direction)))
        {
            if (direction == Direction.None) continue; // Noneはスキップ

            Vector3 offset = direction switch
            {
                Direction.North => new Vector3(0, 0, -1),
                Direction.NorthEast => Vector3.Normalize(new Vector3(1, 0, -1)),
                Direction.East => new Vector3(1, 0, 0),
                Direction.SouthEast => Vector3.Normalize(new Vector3(1, 0, 1)),
                Direction.South => new Vector3(0, 0, 1),
                Direction.SouthWest => Vector3.Normalize(new Vector3(-1, 0, 1)),
                Direction.West => new Vector3(-1, 0, 0),
                Direction.NorthWest => Vector3.Normalize(new Vector3(-1, 0, -1)),
                _ => Vector3.Zero
            };

            // 距離を適用して座標を計算
            Vector3 position = (center ?? new Vector3(100, 0, 100)) + (offset * distance);

            // リストに追加
            directionalVectors.Add(new DirectionalVector(direction, position));
        }

        return directionalVectors;
    }

    // 2つのDirectionを比較して、角度を返す。角度は正しい値ではなく0, 45, 90, 135, 180の値を返す
    private int GetTwoPointAngle(Direction direction1, Direction direction2)
    {
        // enumの値を数値に変換
        int angle1 = (int)direction1;
        int angle2 = (int)direction2;

        // 環状の差分を計算
        int diff = (angle2 - angle1 + 8) % 8; // 環状に補正して差分を取得

        // 差分に応じた角度を計算（時計回りで正、反時計回りで負）
        int angle = diff switch
        {
            0 => 0,
            1 => 45,
            2 => 90,
            3 => 135,
            4 => 180,
            5 => -135,
            6 => -90,
            7 => -45,
            _ => 0 // このケースは通常発生しない
        };

        return angle;
    }

    // 2つのDirectionを比較して、左右どちらかを返す。左なら-1、右なら1、同じまたは逆なら0を返す
    private LR GetTwoPointLeftRight(Direction direction1, Direction direction2)
    {
        if (direction1 == Direction.North && direction2 == Direction.NorthEast) return LR.Right;
        if (direction1 == Direction.NorthEast && direction2 == Direction.East) return LR.Right;
        if (direction1 == Direction.East && direction2 == Direction.SouthEast) return LR.Right;
        if (direction1 == Direction.SouthEast && direction2 == Direction.South) return LR.Right;
        if (direction1 == Direction.South && direction2 == Direction.SouthWest) return LR.Right;
        if (direction1 == Direction.SouthWest && direction2 == Direction.West) return LR.Right;
        if (direction1 == Direction.West && direction2 == Direction.NorthWest) return LR.Right;
        if (direction1 == Direction.NorthWest && direction2 == Direction.North) return LR.Right;

        if (direction1 == Direction.North && direction2 == Direction.West) return LR.Left;
        if (direction1 == Direction.West && direction2 == Direction.South) return LR.Left;
        if (direction1 == Direction.South && direction2 == Direction.East) return LR.Left;
        if (direction1 == Direction.East && direction2 == Direction.North) return LR.Left;

        if (direction1 == Direction.North && direction2 == Direction.SouthEast) return LR.Right;
        if (direction1 == Direction.NorthEast && direction2 == Direction.South) return LR.Right;
        if (direction1 == Direction.East && direction2 == Direction.SouthWest) return LR.Right;
        if (direction1 == Direction.SouthEast && direction2 == Direction.West) return LR.Right;
        if (direction1 == Direction.South && direction2 == Direction.NorthWest) return LR.Right;
        if (direction1 == Direction.SouthWest && direction2 == Direction.North) return LR.Right;
        if (direction1 == Direction.West && direction2 == Direction.NorthEast) return LR.Right;
        if (direction1 == Direction.NorthWest && direction2 == Direction.East) return LR.Right;

        return LR.Left;
    }
    private void HideAllElements() => Controller.GetRegisteredElements().Each(x => x.Value.Enabled = false);

    private Direction Rotate45Clockwise(Direction direction)
    {
        return direction switch
        {
            Direction.North => Direction.NorthEast,
            Direction.NorthEast => Direction.East,
            Direction.East => Direction.SouthEast,
            Direction.SouthEast => Direction.South,
            Direction.South => Direction.SouthWest,
            Direction.SouthWest => Direction.West,
            Direction.West => Direction.NorthWest,
            Direction.NorthWest => Direction.North,
            _ => Direction.None
        };
    }

    private Direction GetOppositeDirection(Direction direction)
    {
        return direction switch
        {
            Direction.North => Direction.South,
            Direction.NorthEast => Direction.SouthWest,
            Direction.East => Direction.West,
            Direction.SouthEast => Direction.NorthWest,
            Direction.South => Direction.North,
            Direction.SouthWest => Direction.NorthEast,
            Direction.West => Direction.East,
            Direction.NorthWest => Direction.SouthEast,
            _ => Direction.None
        };
    }

    private void ApplyElement(string elementName, Direction direction, float radius, float elementRadius = 0.3f)
    {
        var position = new Vector3(100, 0, 100);
        var angle = GetAngle(direction);
        position += radius * new Vector3(MathF.Cos(MathF.PI * angle / 180f), 0, MathF.Sin(MathF.PI * angle / 180f));
        if (Controller.TryGetElementByName(elementName, out var element))
        {
            element.Enabled = true;
            element.radius = elementRadius;
            element.SetOffPosition(position);
        }
    }

    private float GetAngle(Direction direction)
    {
        return direction switch
        {
            Direction.North => 270,
            Direction.NorthEast => 315,
            Direction.East => 0,
            Direction.SouthEast => 45,
            Direction.South => 90,
            Direction.SouthWest => 135,
            Direction.West => 180,
            Direction.NorthWest => 225,
            _ => 0
        };
    }

    /// <summary>
    /// Calculates the correction angle needed to rotate the object to face the target.
    /// </summary>
    /// <param name="origin">The current position of the object.</param>
    /// <param name="target">The position of the target.</param>
    /// <param name="rotation">The current rotation angle of the object (in radian).</param>
    /// <returns>The correction angle (in degrees) needed to face the target.</returns>
    public static float GetCorrectionAngle(Vector3 origin, Vector3 target, float rotation) => GetCorrectionAngle(MathHelper.ToVector2(origin), MathHelper.ToVector2(target), rotation);

    public static float GetCorrectionAngle(Vector2 origin, Vector2 target, float rotation)
    {
        // Calculate the relative angle to the target
        Vector2 direction = target - origin;
        float relativeAngle = MathF.Atan2(direction.Y, direction.X) * (180 / MathF.PI);

        // Normalize relative angle to 0-360 range
        relativeAngle = (relativeAngle + 360) % 360;

        // Calculate the correction angle
        float correctionAngle = (relativeAngle - ConvertRotationRadiansToDegrees(rotation) + 360) % 360;

        // Adjust correction angle to range -180 to 180 for shortest rotation
        if (correctionAngle > 180)
            correctionAngle -= 360;

        return correctionAngle;
    }

    /// <summary>
    /// Converts a rotation angle in radians to degrees in a system where:
    /// - North is 0°
    /// - Angles increase clockwise
    /// - Range is 0° to 360°
    /// </summary>
    /// <param name="radians">The rotation angle in radians.</param>
    /// <returns>The equivalent rotation angle in degrees.</returns>
    public static float ConvertRotationRadiansToDegrees(float radians)
    {
        // Convert radians to degrees with coordinate system adjustment
        float degrees = ((-radians * (180 / MathF.PI)) + 180) % 360;

        // Ensure the result is within the 0° to 360° range
        return degrees < 0 ? degrees + 360 : degrees;
    }

    /// <summary>
    /// Converts a rotation angle in degrees to radians in a system where:
    /// - North is 0°
    /// - Angles increase clockwise
    /// - Range is -π to π
    /// </summary>
    /// <param name="degrees">The rotation angle in degrees.</param>
    /// <returns>The equivalent rotation angle in radians.</returns>
    public static float ConvertDegreesToRotationRadians(float degrees)
    {
        // Convert degrees to radians with coordinate system adjustment
        float radians = -(degrees - 180) * (MathF.PI / 180);

        // Normalize the result to the range -π to π
        radians = ((radians + MathF.PI) % (2 * MathF.PI)) - MathF.PI;

        return radians;
    }
    #endregion
}
```

### JP Box
the setup is as follows, with SW and SE, N and S swapping, and H2 and D4 making adjustments:
```
T1 T2 H1 H2
D1 D2 D3 D4
```
```
T1 - Put No name 
0 = NW // 1 = None // 2 = None

T2 - Put T1 name
0 = S // 1 = NW // 2 = None

H1 - T1 > T2
0 = NE // 1 = S // 2 =  NW

H2 -T1 > T2 > H1
0 = SW // 1 = NE // 2 = S
--------------------------
D1 - Put no name
0 = SE // 1 = None // 2 = None

D2 - Put D1 name
0 = N // 1 = SE // 2 = None

D3- D1 > D2
 0 = SW // 1 = N // 2 = SE

D4 -  D1 > D2 > D3
0 = NE// 1=  SW -// 2 = N
```

Thanks for Tora

## [LR] Light Tether Visual Indicator
```
~Lv2~{"Name":"P2 [LR] Light Tether Visual Indicator","Group":"FRU","ZoneLockH":[1238],"Scenes":[3],"ElementsL":[{"Name":"","type":1,"color":4293721856,"fillIntensity":0.5,"refActorRequireBuff":true,"refActorBuffId":[4157,4157],"refActorComparisonType":4,"onlyTargetable":true,"onlyVisible":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"","type":3,"refZ":0.54,"radius":0.0,"color":4293721856,"fillIntensity":0.5,"thicc":6.0,"refActorRequireBuff":true,"refActorBuffId":[4157,4157],"refActorComparisonType":4,"onlyTargetable":true,"onlyVisible":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}]}
```

## Flower-like explosions
Will show after knockback, to not obstruct your view. Feel free to edit that out but good luck seeing anything...
```
~Lv2~{"Name":"P2 - Flowers explosions","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"前後","type":3,"refY":40.0,"offY":-40.0,"radius":2.0,"color":4278190335,"fillIntensity":0.139,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40200],"refActorUseCastTime":true,"refActorCastTimeMin":3.0,"refActorCastTimeMax":20.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0,"FillStep":2.0},{"Name":"左右","type":3,"refY":40.0,"offY":-40.0,"radius":2.0,"color":4278190335,"fillIntensity":0.144,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40200],"refActorUseCastTime":true,"refActorCastTimeMin":3.0,"refActorCastTimeMax":20.0,"refActorComparisonType":6,"includeRotation":true,"AdditionalRotation":1.5707964,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0,"FillStep":2.0},{"Name":"右斜","type":3,"refY":40.0,"offY":-40.0,"radius":2.0,"color":4278190335,"fillIntensity":0.144,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40200],"refActorUseCastTime":true,"refActorCastTimeMin":3.0,"refActorCastTimeMax":20.0,"refActorComparisonType":6,"includeRotation":true,"AdditionalRotation":0.7853982,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0,"FillStep":2.0},{"Name":"左斜","type":3,"refY":40.0,"offY":-40.0,"radius":2.0,"color":4278190335,"fillIntensity":0.144,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40200],"refActorUseCastTime":true,"refActorCastTimeMin":3.0,"refActorCastTimeMax":20.0,"refActorComparisonType":6,"includeRotation":true,"AdditionalRotation":5.497787,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0,"FillStep":2.0}]}
```

## Axe/Scythe kick
Scythe kick is purple to help quickly differentiate them
```
~Lv2~{"Name":"P2 - Axe/Scythe kick","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"アクスキック","type":1,"radius":16.0,"refActorNPCNameID":13554,"refActorRequireCast":true,"refActorCastId":[40202],"refActorComparisonType":6,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"サイスキック","type":1,"radius":4.0,"Donut":20.0,"color":3371237631,"fillIntensity":0.3,"refActorNPCNameID":13554,"refActorRequireCast":true,"refActorCastId":[40203],"refActorComparisonType":6,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}]}
```

## Knockback safe spot
Saves safe knockback spot for future use. Does NOT resolves specific spot where you need to go for knockback.
```
~Lv2~{"Name":"P2 - first icicle impact","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"","type":1,"radius":4.93,"color":3357277952,"Filled":false,"fillIntensity":0.5,"thicc":7.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"","type":1,"offY":32.0,"radius":4.93,"color":3357277952,"Filled":false,"fillIntensity":0.5,"thicc":7.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Line 1","type":3,"refY":36.0,"offY":-4.0,"radius":0.0,"color":1679163136,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"KB point","type":1,"offY":16.0,"radius":0.0,"color":4279631616,"Filled":false,"fillIntensity":0.5,"thicc":10.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}],"Freezing":true,"FreezeFor":15.0,"IntervalBetweenFreezes":20.0}
```

## Spread/protean diagram
Diagram to indicate valid positions for flowers and proteans. Does NOT resolves your specific position, does NOT accounts for scythe/axe variants. 
```
~Lv2~{"Name":"P2 - first icicle impact - hints","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"Tower 1","type":1,"offX":5.5,"offY":10.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Tower 1","type":1,"offX":-5.5,"offY":10.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Tower 1","type":1,"offX":5.5,"offY":21.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Tower 1","type":1,"offX":-5.5,"offY":21.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Tower 2","type":1,"offX":13.5,"offY":2.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Tower 2","type":1,"offX":-13.5,"offY":2.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Tower 2","type":1,"offX":-13.5,"offY":29.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Tower 2","type":1,"offX":13.5,"offY":29.5,"radius":1.0,"color":1677787125,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Line 1","type":3,"refY":36.0,"offY":-4.0,"radius":0.0,"color":1679163136,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"Line 1","type":3,"refX":-16.0,"refY":20.0,"offX":-16.0,"offY":-20.0,"radius":0.0,"color":1679163136,"Filled":false,"fillIntensity":0.5,"thicc":4.0,"refActorNPCNameID":12809,"refActorRequireCast":true,"refActorCastId":[40198],"refActorUseCastTime":true,"refActorCastTimeMax":1.0,"refActorComparisonType":6,"includeRotation":true,"AdditionalRotation":1.5707964,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}],"Freezing":true,"FreezeFor":8.5,"IntervalBetweenFreezes":20.0}
```

## Usurper or Frost finder
Add that will cast twin stillness/silence
```
~Lv2~{"Name":"P2 - Usurper of Frost finder","Group":"FRU","ZoneLockH":[1238],"DCond":5,"ElementsL":[{"Name":"","type":1,"radius":2.0,"color":3372154890,"fillIntensity":0.5,"thicc":4.0,"refActorName":"*","refActorRequireCast":true,"refActorCastId":[40208],"refActorUseCastTime":true,"refActorCastTimeMin":3.0,"refActorCastTimeMax":23.300001,"refActorUseOvercast":true,"includeRotation":true,"tether":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}]}
```

## Twin stillness / Twin silence
```
~Lv2~{"Name":"P2 - Twin stillness","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"","type":4,"radius":40.0,"coneAngleMin":-145,"coneAngleMax":145,"fillIntensity":0.259,"refActorNPCNameID":13554,"refActorRequireCast":true,"refActorCastId":[40193],"refActorUseCastTime":true,"refActorCastTimeMax":3.0,"refActorUseOvercast":true,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"","type":4,"radius":40.0,"coneAngleMin":-45,"coneAngleMax":45,"fillIntensity":0.259,"refActorNPCNameID":13554,"refActorRequireCast":true,"refActorCastId":[40193],"refActorUseCastTime":true,"refActorCastTimeMin":3.0,"refActorCastTimeMax":6.0,"refActorUseOvercast":true,"refActorComparisonType":6,"includeRotation":true,"AdditionalRotation":3.1415927,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}]}
```
```
~Lv2~{"Name":"P2 - Twin silence","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"","type":4,"radius":40.0,"coneAngleMin":-145,"coneAngleMax":145,"fillIntensity":0.259,"refActorNPCNameID":13554,"refActorRequireCast":true,"refActorCastId":[40194],"refActorUseCastTime":true,"refActorCastTimeMin":3.0,"refActorCastTimeMax":6.0,"refActorUseOvercast":true,"refActorComparisonType":6,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"","type":4,"radius":40.0,"coneAngleMin":-45,"coneAngleMax":45,"fillIntensity":0.259,"refActorNPCNameID":13554,"refActorRequireCast":true,"refActorCastId":[40194],"refActorUseCastTime":true,"refActorCastTimeMax":3.0,"refActorUseOvercast":true,"refActorComparisonType":6,"includeRotation":true,"AdditionalRotation":3.1415927,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}]}
```

## [Partially EN] Twin stillness/silence KB helper
Shows early only for EN clients, requires translation
```
~Lv2~{"Name":"P2 - Twin silence/stillness KB helper","Group":"FRU","ZoneLockH":[1238],"DCond":5,"ElementsL":[{"Name":"","type":3,"refY":32.0,"radius":0.0,"color":3356425984,"fillIntensity":0.345,"thicc":4.0,"refActorType":1,"includeRotation":true,"LineEndA":1,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}],"UseTriggers":true,"Triggers":[{"Type":2,"Duration":6.0,"Match":">40193)"},{"Type":2,"Duration":6.0,"Match":">40194)"},{"Type":2,"Duration":5.0,"MatchIntl":{"En":"The Light shall usher in peace!"},"MatchDelay":3.1}]}
```

## [EN] Shining armor helper
Don't look.
Shows only for EN clients, requires translation
```
~Lv2~{"Name":"P2 - Don't look","Group":"FRU","ZoneLockH":[1238],"DCond":5,"ElementsL":[{"Name":"","type":4,"refY":10.0,"radius":20.0,"coneAngleMin":-46,"coneAngleMax":46,"color":3356425984,"fillIntensity":0.1,"overlayBGColor":4278190335,"overlayTextColor":4294967295,"overlayVOffset":3.0,"overlayText":"Don't intersect red!","refActorType":1,"includeRotation":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"","type":1,"radius":0.0,"refActorType":1,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"","type":1,"radius":0.0,"Filled":false,"fillIntensity":0.5,"overlayBGColor":4278190080,"overlayTextColor":4293328640,"overlayFScale":3.0,"thicc":4.0,"overlayText":"!!! Don't LOOK !!!","refActorDataID":17823,"refActorComparisonType":3,"includeRotation":true,"tether":true,"LimitDistance":true,"LimitDistanceInvert":true,"DistanceSourceX":100.0,"DistanceSourceY":100.0,"DistanceMax":18.0,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"","type":1,"radius":0.0,"Filled":false,"fillIntensity":0.5,"overlayBGColor":4278190080,"overlayTextColor":4293328640,"overlayFScale":3.0,"thicc":4.0,"overlayText":"!!! Don't LOOK !!!","refActorDataID":17824,"refActorComparisonType":3,"includeRotation":true,"tether":true,"LimitDistance":true,"LimitDistanceInvert":true,"DistanceSourceX":100.0,"DistanceSourceY":100.0,"DistanceMax":18.0,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}],"UseTriggers":true,"Triggers":[{"Type":2,"Duration":3.2,"MatchIntl":{"En":"The Light shall usher in peace!"},"MatchDelay":2.0}]}
```

## Reflected scythe kick
```
~Lv2~{"Name":"P2 - Reflected scythe kick","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"","type":1,"radius":4.0,"Donut":10.0,"color":3371237631,"fillIntensity":0.292,"refActorNPCNameID":9317,"refActorRequireCast":true,"refActorCastId":[40205],"refActorComparisonType":6,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}]}
```

## Banish 3
```
~Lv2~{"Name":"P2 - Banish III Pair","Group":"FRU","ZoneLockH":[1238],"DCond":5,"ElementsL":[{"Name":"二","type":1,"radius":5.5,"color":4278190335,"Filled":false,"fillIntensity":0.3,"overlayBGColor":4278190080,"overlayTextColor":4294967295,"overlayVOffset":2.0,"overlayFScale":2.0,"thicc":5.0,"overlayText":"Pair","refActorComparisonType":1,"refActorType":1,"includeRotation":true,"FaceMe":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}],"UseTriggers":true,"Triggers":[{"Type":2,"Duration":5.5,"Match":"(12809>40220)"}]}
```
```
~Lv2~{"Name":"P2 - Banish III Spread","Group":"FRU","ZoneLockH":[1238],"DCond":5,"ElementsL":[{"Name":"近くにいるプレイヤー","type":1,"radius":6.0,"color":4278190335,"fillIntensity":0.3,"overlayBGColor":4278190080,"overlayTextColor":4294967295,"overlayVOffset":2.0,"overlayFScale":2.0,"refActorPlaceholder":["<2>","<3>","<4>","<5>","<6>","<7>","<8>"],"refActorComparisonType":5,"includeRotation":true,"FaceMe":true,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0},{"Name":"ー","type":1,"radius":0.0,"Filled":false,"fillIntensity":0.5,"overlayBGColor":4278190080,"overlayTextColor":4294967295,"overlayVOffset":2.0,"overlayFScale":2.0,"thicc":0.0,"overlayText":"Spread","refActorType":1,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}],"UseTriggers":true,"Triggers":[{"Type":2,"Duration":5.5,"Match":"(12809>40221)"}],"MaxDistance":7.5,"UseDistanceLimit":true,"DistanceLimitType":1}
```

## Light rampant orb explosion
```
~Lv2~{"Name":"P2 - Light rampant orb explosion","Group":"FRU","ZoneLockH":[1238],"ElementsL":[{"Name":"","type":1,"radius":11.0,"refActorNPCNameID":9318,"refActorRequireCast":true,"refActorCastId":[40219],"refActorUseCastTime":true,"refActorCastTimeMin":2.0,"refActorCastTimeMax":4.7,"refActorComparisonType":6,"refActorTetherTimeMin":0.0,"refActorTetherTimeMax":0.0}]}
```

## [Script] Intermission ice deleter
Deletes annoying white ice from intermission, fixing bug which renders AOEs invisible and making your eyes hurt a lot less on that phase. DO NOT GO TO MIDDLE.
```
https://github.com/PunishXIV/Splatoon/raw/refs/heads/main/SplatoonScripts/Duties/Dawntrail/The%20Futures%20Rewritten/P2%20Delete%20Intermission%20Ice.cs
```

## [Beta] [Script] AutoTargetCrystal
Auto-targets nearest light crystal, then auto-targets the center crystal after all light crystals are destroyed.
No configuration needed.
```
https://github.com/PunishXIV/Splatoon/raw/main/SplatoonScripts/Duties/Dawntrail/The%20Futures%20Rewritten/P2%20AutoTargetCrystal.cs
```
