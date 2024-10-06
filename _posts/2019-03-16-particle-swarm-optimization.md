---
layout: post
title:  "Particle Swarm Optimization in C#"
date:   2019-03-19 15:00:00 +0200
#last_modified_at:   2019-01-02 17:00:00 +0200
feature_image: "https://unsplash.it/1200/400?image=225"
category: Development
tags: [csharp, c#, particle swarm optimization, pso]
published: false
---

I implemented a simple particle swarm optimizer in C# to help me optimize my
trading systems.

<!-- more -->

Particle Swarm Optimization (PSO) is a simple computational method to optimize a
problem by simulating a set of moving particles that move around a search-space.
Each particle has a position and velocity and in each step the particle fitness
is calculated and the particle is moved to a new position that depends on its
current position and the best found position so far.After some iterations the
best particle position converges to a solution.

The following example shows how the C# code can be used:

{% highlight csharp %}
var lowerBound = new double[] { 0.0, 0.0 };
var upperBound = new double[] { 1.0, 1.0 };
var swarm = new ParticleSwarm(20, lowerBound, upperBound, p =>
{
  return Math.Cos(p[0]) + Math.Sin(p[1]);
});

swarm.Step(100, i => {
  Console.WriteLine($"{i}; {swarm.BestFitness}; {swarm.BestPosition[0]}; {swarm.BestPosition[1]}");
  return Console.KeyAvailable;
});
{% endhighlight %}

The code sets up the boundaries of the search-space and specifies the function
to be optimized. After that the particle swarm is stepped for at most 100
iterations and in each iteration the currenly best found solution is printed to
the console.

The full source to the particle swarm follows below. It can also be found on my
github repository ([here](https://github.com/trenki2/ParticleSwarmOptimization)).

{% highlight csharp %}
using System;
using System.Threading.Tasks;

namespace ParticleSwarmOptimization
{
  public class ParticleSwarm
  {
    // Particle swarm parameters. 
    // https://en.wikipedia.org/wiki/Particle_swarm_optimization
    public double Omega { get; set; } = 0.729;
    public double Phi_G { get; set; } = 1.49445;
    public double Phi_P { get; set; } = 1.49445;

    /// <summary>
    /// Gets the best fitness of all particles.
    /// </summary>
    /// <value>The best fitness.</value>
    public double BestFitness { get; private set; }

    /// <summary>
    /// Gets the best position of all particles.
    /// </summary>
    /// <value>The best position.</value>
    public double[] BestPosition { get; private set; }

    private class Particle
    {
      public double[] position;
      public double[] velocity;
      public double[] bestPosition;
      public double fitness;
      public double bestFitness;

      public Particle(int numDimensions)
      {
        position = new double[numDimensions];
        velocity = new double[numDimensions];
        bestPosition = new double[numDimensions];
        bestFitness = fitness = -double.MaxValue;
      }
    }

    private Random random = new Random();
    private Particle[] particles;

    private double[] lowerBound;
    private double[] upperBound;
    private Func<double[], double> fitnessFunc;

    /// <summary>
    /// Initializes a new instance of the <see cref="T:ParticleSwarmOptimization.ParticleSwarm"/> class.
    /// The particle swarm maximizes the particle fitness.
    /// </summary>
    /// <param name="numParticles">Number of particles.</param>
    /// <param name="lowerBound">Lower bound of the solution space.</param>
    /// <param name="upperBound">Upper bound of the solution space.</param>
    /// <param name="evalFunc">Evaluation function which takes the particle positions and returns its fitness.</param>
    public ParticleSwarm(int numParticles, double[] lowerBound, double[] upperBound, Func<double[], double> evalFunc)
    {
      if (lowerBound.Length != upperBound.Length)
        throw new ArgumentException("Dimensions of lower and upper bound do not match");

      this.fitnessFunc = evalFunc;
      this.lowerBound = lowerBound;
      this.upperBound = upperBound;

      var numDimensions = lowerBound.Length;

      BestFitness = -double.MaxValue;
      BestPosition = new double[numDimensions];
      particles = new Particle[numParticles];

      for (int i = 0; i < numParticles; i++)
      {
          var p = new Particle(numDimensions);    

          for (int j = 0; j < numDimensions; j++)
          {
              var diff = upperBound[j] - lowerBound[j];
              p.position[j] = NextDoubleInRange(lowerBound[j], upperBound[j]);
              p.velocity[j] = NextDoubleInRange(-diff, +diff);
          }

          particles[i] = p;
      }
    }

    private double NextDoubleInRange(double min, double max)
    {
        return random.NextDouble() * (max - min) + min;
    }

    /// <summary>
    /// Step the particle swarm for a given number of steps.
    /// </summary>
    /// <param name="count">Maximum number of steps.</param>
    /// <param name="stepFunc">Step function. Takes current iteration counter and returns true when the stepping should be aborted.</param>
    public void Step(int count, Func<int, bool> stepFunc)
    {
      for (int l = 0; l < count; l++)
      {
        Parallel.For(0, particles.Length, i =>
        {
          EvaluateParticle(particles[i]);
          MoveParticle(particles[i]);
        });

        if (stepFunc(l))
            break;
      }
    }

    private void EvaluateParticle(Particle p)
    {
      p.fitness = fitnessFunc(p.position);

      if (p.fitness > p.bestFitness)
      {
        p.bestFitness = p.fitness;
        Array.Copy(p.position, p.bestPosition, p.position.Length);

        lock (this)
        {
          if (p.bestFitness > BestFitness)
          {
            BestFitness = p.bestFitness;
            Array.Copy(p.bestPosition, BestPosition, p.bestPosition.Length);
          }
        }
      }
    }

    private void MoveParticle(Particle p)
    {
      for (int i = 0; i < p.position.Length; i++)
      {
        var rp = random.NextDouble();
        var rg = random.NextDouble();

        p.velocity[i] = Omega * p.velocity[i]
            + Phi_P * rp * (p.bestPosition[i] - p.position[i])
            + Phi_G * rg * (BestPosition[i] - p.position[i]);

        p.position[i] += p.velocity[i];

        if (p.position[i] > upperBound[i])
          p.position[i] = upperBound[i];
        if (p.position[i] < lowerBound[i])
          p.position[i] = lowerBound[i];
      }
    }
  }
}
{% endhighlight %}