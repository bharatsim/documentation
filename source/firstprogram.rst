Writing your First Program
==========================

The SIR Model in BharatSim
--------------------------

BharatSim is an open-source large-scale agent-based simulation framework that allows you to build and simulate sophisticated agent-based models. Agents form the main component of such a simulation. They are entities that are defined with a set of parameters and obey certain specified conditions as the simulation runs.

In the purview of the SIR model, the agents are people, which go about their daily lives and follow regular schedules. Here, the parameters defining an agent are a unique agent id, their age, their infection status, the time for which they remain infected and so on. The framework allows you to introduce new parameters as and when required.

The rules governing the behaviour of all agents include the particular schedules which they follow and the functions which allow the agents to transit from one compartment to another as described before.

Model Components
~~~~~~~~~~~~~~~~

The model consists of different objects and classes and can be separated into individual ``.scala`` files.

#. **Nodes of the Network:** These include ``House``, ``School``, ``Office``, etc, which essentially correspond to the different geographical locations in which the agents spend their time according to some schedule. These classes have some attributes associated with them. For eg. ``getContactProbability`` is one such function that gives the transmission probability of the disease in that particular location. In order to add new Nodes to the Network (for eg. Hospital), one has to create a new scala class inside the same package corresponding to the new Node.

    .. image:: _static/images/House.png

#. **Disease:** ``Disease`` is a class where all epidemiological parameters are conveniently defined. For example, the transmission rate :math:`\beta` is defined in this class. This class allows for a convenient way to store and modify variables as required.

#. **Infection Status:** The ``InfectionStatus`` is the class where all the compartments of the model are defined. The ``InfectionStatus`` object in the class, is a serialisable object which includes all the compartments. Inside this class, there is an encoder function which converts the compartment into a corresponding string and a decoder function, which decodes a string into a particular compartment. To add a new compartment like ‘Exposed’ refer to the figure below:-

    .. code-block:: scala

      package com.bharatsim.examples.epidemiology.sir

      import com.bharatsim.engine.basicConversions.StringValue
      import com.bharatsim.engine.basicConversions.decoders.BasicDecoder
      import com.bharatsim.engine.basicConversions.encoders.BasicEncoder

      object InfectionStatus extends Enumeration {
        type InfectionStatus = Value
        val Susceptible,Exposed, Infected, Removed = Value

        implicit val infectionStatusDecoder: BasicDecoder[InfectionStatus] = {
            case StringValue(v) => withName(v)
            case _ => throw new RuntimeException("Infection status was not stored as a string")
        }

        implicit val infectionStatusEncoder: BasicEncoder[InfectionStatus] = {
            case Susceptible => StringValue("Susceptible")
            case Exposed => StringValue("Exposed")
            case Infected => StringValue("Infected")
            case Removed => StringValue("Removed")
        }
      }

#. **Output Specification:** ``CSVSpecs`` is a framework defined trait which helps in generating a .csv file for the simulation output. The ``SEIROutputSpec`` is a class that extends this trait and defines how the output ``.csv`` file should look. Inside this class, the ``getHeaders`` basically specifies the list of column headers in the ``.csv`` file, while the ``getRows`` fetches the values of these quantities from context. For eg. one can use the ``fetchcount`` function in the framework defined trait ``graphProvider`` to display the number of people in each compartment at every tick.

Person class
~~~~~~~~~~~~

The ``Person`` class is an extension of the framework defined ``Agent`` class. It defines all the attributes of an agent, and also the behaviours which the agent follows throughout the simulation.

An attribute is essentially a characteristic of an agent like age, agent id and so on. It can also include certain quantities associated with an agent like its current epidemiological state, or the time for which it is infected. The attributes are defined in the Person class as follows:-

.. code-block:: scala

   case class Person(id: Long, age: Int, infectionState: InfectionStatus, infectionDur: Int, betaMultiplier:Double) extends Agent

Some attributes have constant values throughout the simulation like age, agent id, while there are some attributes which are ‘dynamic’ and their values change as the simulation runs.

In this framework, A **Behaviour** is an user-defined function whose arguments are the dynamic attributes. It acts on every agent at each tick.  An example for a behaviour which counts the number of days for which an agent is infected, is as follows:

.. code-block:: scala

   private val incrementInfectionDay: Context => Unit = (context: Context) => {
      if (isInfected && context.getCurrentStep % Disease.numberOfTicksInADay == 0) {
         updateParam("infectionDur", infectionDur + 1)
      }
   }

Moreover, a Behaviour could be a rule which governs the transfer of agents from one compartment to another. For example, if person X is in the same location as an infected person Y, person X will also get infected, with some probability.
Similarly, if a person is infected, they can move to the ``Removed`` compartment, with a certain rate, :math:`\lambda_I`. The code snippet below is a Behaviour ``checkForRecovery`` that performs this transition.

.. code-block:: scala

   private val checkForRecovery: Context => Unit = (context: Context) => {
       if (isInfected) {
           val RecoveryProb = Disease.lambdaI*Disease.dt
           val InfectionState = if (biasedCoinToss(RecoveryProb)) "Removed" else "Infected"
           if (InfectionState == "Removed") {
                updateParam("infectionState", Removed)
           }
       }
   }


It is important that one defines the Behaviours in the order that they are to be compiled. This is based on the epidemiological SIR model, where an agent transitions from one compartment to another in a chronological manner, from S to I to R, as shown below.

.. image:: _static/images/SIR-compartment.png

After the behaviours are defined, they need to be added to the simulation in the order in which they are executed. This is done using the framework defined ``addBehaviour`` function.

.. code-block:: scala

   addBehaviour(incrementInfectionDay)
   addBehaviour(checkForInfection)
   addBehaviour(checkForRecovery)

Each agent goes through the above-mentioned behaviours chronologically during each tick. For example, if there are 100 agents in the simulation, all 100 of them go through the behaviours as listed chronologically above, and this process repeats at each tick.


Saving your output
^^^^^^^^^^^^^^^^^^

Suppose you wanted your output to give you the numbers of susceptible, infected and recovered people at every time step. You can then write the following:

.. code-block:: scala

  import com.bharatsim.engine.Context
  import com.bharatsim.engine.graph.patternMatcher.MatchCondition._
  import com.bharatsim.engine.listeners.CSVSpecs
  import com.bharatsim.examples.epidemiology.SIR.InfectionStatus.{Infected, Removed, Susceptible}
  
  class SIROutputSpec(context: Context) extends CSVSpecs {
    override def getHeaders: List[String] =
      List(
        "Step",
        "Susceptible",
        "Infected",
        "Removed"
      )
  
    override def getRows(): List[List[Any]] = {
      val graphProvider = context.graphProvider
      val label = "Person"
      val row = List(
        context.getCurrentStep,
        graphProvider.fetchCount(label, "infectionState" equ Susceptible),
        graphProvider.fetchCount(label, "infectionState" equ Infected),
        graphProvider.fetchCount(label, "infectionState" equ Removed)
      )
      List(row)
    }
  }
 
* The first column (Step) stores the current time step, obtained using the ``context.getCurrentStep`` function
* The next 3 columns store the number of Susceptible, Infected and Removed people respectively, by fetching the total number of ``Person`` nodes on the graph with the appropriate Infection Status.

Now we simply have to register it in the simulation. Note that the following code snippet should be located inside ``simulation.defineSimulation`` in the main function:

.. code-block:: scala

  SimulationListenerRegistry.register(
    new CsvOutputGenerator("src/main/resources/output.csv", new SIROutputSpec(context))
      )