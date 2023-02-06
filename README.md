# what is that?
This project is an attempt to create fast-paced strategy game in VR, it was created as the final project for the VR Diploma of the US embassy in Egypt.
# The Architecture
## 1. Client-Server Simulation
### Simulate or Send the final data?
One thing I don't like about server authoritative coding is that we have to choose what data to be transferred to the client, and what the client can simulate on it's own, for example if we have rules for time taken to harvest a farm based on the game context, the client already have the same context and the action is deterministic and not affected by the other incoming server calls. So we have to duplicate the same logic twice, in the server and on the client.
### Simlulate All
Actually, the ideal way to make the client responsive, send less data, and validate everything on the server is to have the logic on a layer shared between the client and the server, Unity and ASP.NET Core in this case. So I created a class library following the .NET Standard 2.1, which Unity supports.<br />
The state of the shared logic should get mutated on the client and the server by the player commands and the automatic actions. So the logic layer must produce the same state with the same set of inputs. Non-deterministic actions requires the client await the server result. An example of the such action is a simple attack action, you may initiate an attack while your turn is not over locally, but you have 600ms latency which exceeds the allowance by the server, so the server will reject the action naturally, so before the client continues with mutating the logical layer locally is should assure the server result call was successful.
### The logic layer and MonoBehaviours
The real problem comes after that, how to create and use the shared layer API. This is a problem because in Unity, most of the game classes are `MonoBehaviour` for several reasons, so you can't extend the functionality by inheritance, you will use composition instead, and this deprives us from the "is a" relationship benefits.<br />
One example of this is the slot class, a slot is the location where you can place the units. The shared library has slot class for , and the client must use it as base for its logic. So it's used as a property in a new class called `SlotBehaviour` that is inheriting from `MonoBehaviour` and attached to the slot prefabs, that slot property is used to mutate the state of the shared logic system, it informs the server first was the done commands.
When the `MonoBehaviour` is not required, you can inherit from the class directly, and use all the goods of inheritance stuff like virtual and abstract memebers. Some times it makes more since to extended the functionality by inheritance then create a behaviour type that use it, for example create `Slot`, `ClientSlot`, and `SlotBehaviour`.
``` C#
 public class Slot
    {
        public (int x, int z) Index { get; }
        public FieldUnit Unit { get; set; }
        public RoomActorCore RoomActorCore { get; }

        public Slot((int x, int z) index, RoomActorCore roomActorCore)
        {
            Index = index;
            RoomActorCore = roomActorCore;
        }

        public TUnit CreateBloodUnit<TUnit>() where TUnit : FieldUnit
        {
            var newUnit = FluentUtility.I.CreateFieldUnit<TUnit>(this);

            Unit = newUnit;
            RoomActorCore.Board.CheckHead(newUnit);

            return newUnit;
        }
    }
}
```

This shows the usage of the inner logic objects wrapped around MonoBehaviours, See how the operation is assured from the server first.
``` C#
public class SlotBehaviour : InteractableBehaviour, IClickable, IHoverable
{
    public Slot Slot;
    public RoomUser RoomUser;

  /// <summary>
    /// used by the local placeable object which has  already instantiated the right game object
    /// </summary>
    public async UniTask<bool> CreateFieldUnit<TLogic>(FieldUnitBehaviour<TLogic> unit) where TLogic : FieldUnit
    {
        var feasible = await NetManager.I.InvokeAsync<bool>("PlaceUnit", typeof(TLogic).Name, Slot.Index);
        if (!feasible)
        {
            Debug.LogError("the operation was not feasible from the server");
            return false;
        }

        var logicUnit = Slot.CreateBloodUnit<TLogic>();

        unit.Init(logicUnit, RoomUser);
        unit.SlotBehaviour = this;

        unitBehaviour = unit;

        SpatiallyPlaceUnit(unit);
        return true;
    }
}
```

## Create unit of anytime at runtime dynamically
``` C#
public static class ActivatorUtils
    {
        public delegate object ObjectActivator<T>(params object[] args);
        //this is just a declaring type signature

        public static ObjectActivator<T> GetActivator<T>(ConstructorInfo ctor)
        {
            var paramsInfo = ctor.GetParameters();

            //create a single param of type object[], to pass to ObjectActivator delegate
            var objArrParamExp = Expression.Parameter(typeof(object[]), "args");

            var argsExps = new Expression[paramsInfo.Length];

            for (var i = 0; i < paramsInfo.Length; i++)
            {
                Expression index = Expression.Constant(i); //creates a const of int i

                var paramType = paramsInfo[i].ParameterType;

                Expression paramAccessorExp = Expression.ArrayIndex(objArrParamExp, index);
                //access parameter with index in the array expression
                //why to use obj[]? because we want to save them in the ObjectActivator with obj[] as input

                Expression paramCastExp = Expression.Convert(paramAccessorExp, paramType);
                //convert the given param to the type from the given constructor param list

                argsExps[i] = paramCastExp;
                //expression for each parameter: access array object, cast it
            }

            //make a NewExpression that calls the ctor with the args we just created
            var newExp = Expression.New(ctor, argsExps);

            //create a lambda with the New Expression as body and our param object[] as arg
            var lambda = Expression.Lambda(typeof(ObjectActivator<T>), newExp, objArrParamExp);

            //compile it
            var compiled = (ObjectActivator<T>)lambda.Compile();
            return compiled;
        }
    }
```
Make a list of the delegates with a type-delegate dictionary to use later.
## 2. Composition Over Inheritance
## the unit system
The game consists of units, attacker units like cannons magician, defender units, resource units, etc.. They share functionality, but we can't use inheritance for that! The inheritance tree will have conflicts because of the shared functionality doesn't have to follow the tree, the soldier and the wall units can defened, but the soldier can attack as well. If the tree branched from the unit class to the attacker and defender classes we won't have the ability to have a class that has the functionality of both. So instead We will use a component system to extend the units
``` C#
public abstract class Unit
    {
        protected List<Component> Components { get; set; }

        public T GetComponent<T>() where T : Component
        {
            return (T)Components.FirstOrDefault(c => c is T);
        }

        public bool HasComponent<T>() where T : Component
        {
            return Components.Any(c => c is T);
        }

        public abstract void KillUnit();
    }
```
At the top of the pyramid the usable, sealed Units:
``` C#
public sealed class PurpleMagician : FieldUnit, ISoldier, IPickyAttackerUnit<BloodUnit>
{
    public PurpleMagician(Slot slot) : base(slot)
    {
        BloodDefender = new(this, 150, 1);
        Alive = new(this, 50, TimeSpan.FromSeconds(3));
        Worth = new(this, 500, (Material.Elixir, 70));
        Attacker = new(this, 200, TimeSpan.FromSeconds(5));

        Components = new()
        {
            BloodDefender,
            Alive,
            Worth,
            Attacker,
        };
    }

    public override BloodDefender BloodDefender { get; }
    public override Worth Worth { get; }
    public Alive Alive { get; }
    public PickyAttacker<BloodUnit> Attacker { get; }
}
```
The inheritance hierarchy is very complex to make it future proof for changes with minimal effort, making adding new units very easy.