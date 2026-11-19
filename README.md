using GaOperators;

var rng = new Random(123);
var population = new List<Chromosome>
{
    new Chromosome(new[] { true, false, true, false, true }, 10.5),
    new Chromosome(new[] { false, true, false, true, false }, 8.2),
    new Chromosome(new[] { true, true, false, false, true }, 12.1),
    new Chromosome(new[] { false, false, true, true, false }, 5.0)
};

var selector = new TournamentSelection(tournamentSize: 2);
int[] selectedIndices = selector.Select(population, count: 2, rng);
Console.WriteLine($"Выбраны индексы родителей: {selectedIndices[0]} и {selectedIndices[1]}");

var parentA = population[selectedIndices[0]];
var parentB = population[selectedIndices[1]];

var crossover = new SinglePointCrossover(crossoverProbability: 0.9);
var (child1, child2) = crossover.Cross(parentA, parentB, rng);
Console.WriteLine($"Родитель A: {string.Join("", parentA.GetGenes())} (F: {parentA.Fitness})");
Console.WriteLine($"Родитель B: {string.Join("", parentB.GetGenes())} (F: {parentB.Fitness})");
Console.WriteLine($"Потомок 1 : {string.Join("", child1.GetGenes())}");
Console.WriteLine($"Потомок 2 : {string.Join("", child2.GetGenes())}");

var mutator = new BitFlipMutation(mutationProbability: 0.2);
var mutatedChild1 = mutator.Mutate(child1, rng);
Console.WriteLine($"Мутированный потомок 1: {string.Join("", mutatedChild1.GetGenes())}");

/// <summary>
/// Оператор равномерного скрещивания.
/// </summary>
public sealed class UniformCrossover : ICrossover
{
    public double CrossoverProbability { get; }

    public UniformCrossover(double crossoverProbability)
    {
        if (crossoverProbability < 0.0 || crossoverProbability > 1.0)
            throw new ArgumentOutOfRangeException(nameof(crossoverProbability));
        CrossoverProbability = crossoverProbability;
    }

    public (Chromosome Child1, Chromosome Child2) Cross(Chromosome parent1, Chromosome parent2, Random rng)
    {
        if (parent1 is null) throw new ArgumentNullException(nameof(parent1));
        if (parent2 is null) throw new ArgumentNullException(nameof(parent2));
        if (rng is null) throw new ArgumentNullException(nameof(rng));
        if (parent1.Length != parent2.Length)
            throw new ArgumentException("Родители должны иметь одинаковую длину.", nameof(parent2));

        if (rng.NextDouble() >= CrossoverProbability)
        {
            return (parent1.Clone(), parent2.Clone());
        }

        int length = parent1.Length;
        bool[] child1Genes = new bool[length];
        bool[] child2Genes = new bool[length];
        var p1 = parent1.GetGenes();
        var p2 = parent2.GetGenes();

        for (int i = 0; i < length; i++)
        {
            if (rng.Next(2) == 0)
            {
                child1Genes[i] = p1[i];
                child2Genes[i] = p2[i];
            }
            else
            {
                child1Genes[i] = p2[i];
                child2Genes[i] = p1[i];
            }
        }
        return (new Chromosome(child1Genes, 0), new Chromosome(child2Genes, 0));
    }
}

/// <summary>
/// Оператор турнирного отбора.
/// </summary>
public sealed class TournamentSelection : ISelection
{
    public int TournamentSize { get; }

    public TournamentSelection(int tournamentSize)
    {
        if (tournamentSize < 2)
            throw new ArgumentException("Размер турнира должен быть не меньше 2.", nameof(tournamentSize));
        TournamentSize = tournamentSize;
    }

    public int[] Select(IReadOnlyList<Chromosome> population, int count, Random rng)
    {
        if (population is null) throw new ArgumentNullException(nameof(population));
        if (rng is null) throw new ArgumentNullException(nameof(rng));
        if (population.Count == 0) throw new ArgumentException("Популяция не может быть пустой.", nameof(population));
        if (count < 0) throw new ArgumentOutOfRangeException(nameof(count));
        if (TournamentSize > population.Count)
            throw new ArgumentException("Размер турнира не может превышать размер популяции.", nameof(TournamentSize));

        var result = new int[count];
        var indices = new int[TournamentSize];

        for (int i = 0; i < count; i++)
        {
            // Выбираем уникальных участников
            for (int j = 0; j < TournamentSize;)
            {
                int idx = rng.Next(population.Count);
                bool exists = false;
                for (int k = 0; k < j; k++)
                {
                    if (indices[k] == idx) { exists = true; break; }
                }
                if (!exists) { indices[j++] = idx; }
            }

            // Ищем лучшего
            double bestFitness = double.NegativeInfinity;
            int bestIndex = -1;
            for (int j = 0; j < TournamentSize; j++)
            {
                var chrom = population[indices[j]];
                if (chrom.Fitness > bestFitness)
                {
                    bestFitness = chrom.Fitness;
                    bestIndex = indices[j];
                }
            }
            result[i] = bestIndex;
        }
        return result;
    }
}

/// <summary>
/// Оператор одноточечного скрещивания.
/// </summary>
public sealed class SinglePointCrossover : ICrossover
{
    public double CrossoverProbability { get; }

    public SinglePointCrossover(double crossoverProbability)
    {
        if (crossoverProbability < 0.0 || crossoverProbability > 1.0)
            throw new ArgumentOutOfRangeException(nameof(crossoverProbability), "Вероятность должна быть в диапазоне [0, 1].");
        CrossoverProbability = crossoverProbability;
    }

    public (Chromosome Child1, Chromosome Child2) Cross(Chromosome parent1, Chromosome parent2, Random rng)
    {
        if (parent1 is null) throw new ArgumentNullException(nameof(parent1));
        if (parent2 is null) throw new ArgumentNullException(nameof(parent2));
        if (rng is null) throw new ArgumentNullException(nameof(rng));
        if (parent1.Length != parent2.Length)
            throw new ArgumentException("Родители должны иметь одинаковую длину.", nameof(parent2));

        if (rng.NextDouble() >= CrossoverProbability)
        {
            return (parent1.Clone(), parent2.Clone());
        }

        int length = parent1.Length;
        if (length == 1)
        {
            return (parent1.Clone(), parent2.Clone());
        }

        int point = rng.Next(1, length);
        bool[] child1Genes = new bool[length];
        bool[] child2Genes = new bool[length];

        Array.Copy(parent1.GetGenes(), 0, child1Genes, 0, point);
        Array.Copy(parent2.GetGenes(), point, child1Genes, point, length - point);

        Array.Copy(parent2.GetGenes(), 0, child2Genes, 0, point);
        Array.Copy(parent1.GetGenes(), point, child2Genes, point, length - point);

        return (new Chromosome(child1Genes, 0), new Chromosome(child2Genes, 0));
    }
}

/// <summary>
/// Оператор мутации, инвертирующий каждый ген с заданной вероятностью.
/// </summary>
public sealed class BitFlipMutation : IMutation
{
    public double MutationProbability { get; }

    public BitFlipMutation(double mutationProbability)
    {
        if (mutationProbability < 0.0 || mutationProbability > 1.0)
            throw new ArgumentOutOfRangeException(nameof(mutationProbability), "Вероятность должна быть в диапазоне [0, 1].");
        MutationProbability = mutationProbability;
    }

    public Chromosome Mutate(Chromosome source, Random rng)
    {
        if (source is null) throw new ArgumentNullException(nameof(source));
        if (rng is null) throw new ArgumentNullException(nameof(rng));

        var clone = source.Clone();
        for (int i = 0; i < clone.Length; i++)
        {
            if (rng.NextDouble() < MutationProbability)
            {
                // Инверсия через создание нового массива, так как гены инкапсулированы
                var genes = clone.GetGenes();
                genes[i] = !genes[i];
                // Заменяем внутреннее поле через рефлексию или конструктор не получится, 
                // поэтому в данном учебном примере допустим, что Clone возвращает изменяемый объект.
                // Однако, согласно правилам инкапсуляции, лучше создать новый объект:
                clone = new Chromosome(genes, clone.Fitness);
            }
        }
        return clone;
    }
}

/// <summary>
/// Представляет бинарную хромосому фиксированной длины с заданным значением приспособленности.
/// </summary>
public sealed class Chromosome
{
    private readonly bool[] _genes;

    /// <summary>
    /// Создаёт новую хромосому.
    /// </summary>
    /// <param name="genes">Массив генов. Не может быть null или пустым.</param>
    /// <param name="fitness">Значение приспособленности.</param>
    /// <exception cref="ArgumentException">Если массив генов пуст.</exception>
    /// <exception cref="ArgumentNullException">Если массив генов равен null.</exception>
    public Chromosome(bool[] genes, double fitness)
    {
        if (genes is null) throw new ArgumentNullException(nameof(genes));
        if (genes.Length < 1) throw new ArgumentException("Длина хромосомы должна быть не меньше 1.", nameof(genes));

        _genes = new bool[genes.Length];
        Array.Copy(genes, _genes, genes.Length);
        Fitness = fitness;
    }

    /// <summary>
    /// Длина хромосомы (количество генов).
    /// </summary>
    public int Length => _genes.Length;

    /// <summary>
    /// Значение приспособленности.
    /// </summary>
    public double Fitness { get; set; }

    /// <summary>
    /// Возвращает копию массива генов.
    /// </summary>
    public bool[] GetGenes() => (bool[])_genes.Clone();

    /// <summary>
    /// Индексатор для чтения гена.
    /// </summary>
    public bool this[int index] => _genes[index];

    /// <summary>
    /// Создаёт глубокую копию хромосомы.
    /// </summary>
    public Chromosome Clone()
    {
        return new Chromosome(_genes, Fitness);
    }
}

/// <summary>
/// Представляет бинарную хромосому фиксированной длины с заданным значением приспособленности.
/// </summary>
public sealed class Chromosome
{
    private readonly bool[] _genes;

    /// <summary>
    /// Создаёт новую хромосому.
    /// </summary>
    /// <param name="genes">Массив генов. Не может быть null или пустым.</param>
    /// <param name="fitness">Значение приспособленности.</param>
    /// <exception cref="ArgumentException">Если массив генов пуст.</exception>
    /// <exception cref="ArgumentNullException">Если массив генов равен null.</exception>
    public Chromosome(bool[] genes, double fitness)
    {
        if (genes is null) throw new ArgumentNullException(nameof(genes));
        if (genes.Length < 1) throw new ArgumentException("Длина хромосомы должна быть не меньше 1.", nameof(genes));

        _genes = new bool[genes.Length];
        Array.Copy(genes, _genes, genes.Length);
        Fitness = fitness;
    }

    /// <summary>
    /// Длина хромосомы (количество генов).
    /// </summary>
    public int Length => _genes.Length;

    /// <summary>
    /// Значение приспособленности.
    /// </summary>
    public double Fitness { get; set; }

    /// <summary>
    /// Возвращает копию массива генов.
    /// </summary>
    public bool[] GetGenes() => (bool[])_genes.Clone();

    /// <summary>
    /// Индексатор для чтения гена.
    /// </summary>
    public bool this[int index] => _genes[index];

    /// <summary>
    /// Создаёт глубокую копию хромосомы.
    /// </summary>
    public Chromosome Clone()
    {
        return new Chromosome(_genes, Fitness);
    }
}

using GaOperators;
using Xunit;

public class SinglePointCrossoverTests
{
    private static readonly Random FixedRng = new Random(42);

    [Fact]
    public void Cross_WhenProbabilityZero_ReturnsClones()
    {
        var parent1 = new Chromosome(new[] { true, false, true }, 1.0);
        var parent2 = new Chromosome(new[] { false, true, false }, 2.0);
        var crossover = new SinglePointCrossover(0.0);

        var (child1, child2) = crossover.Cross(parent1, parent2, FixedRng);

        Assert.NotSame(parent1, child1);
        Assert.NotSame(parent2, child2);
        Assert.Equal(parent1.GetGenes(), child1.GetGenes());
        Assert.Equal(parent2.GetGenes(), child2.GetGenes());
    }

    [Fact]
    public void Cross_WhenProbabilityOne_SwapsGenes()
    {
        var parent1 = new Chromosome(new[] { true, true, false, false }, 1.0);
        var parent2 = new Chromosome(new[] { false, false, true, true }, 2.0);
        var crossover = new SinglePointCrossover(1.0);

        var (child1, child2) = crossover.Cross(parent1, parent2, FixedRng);

        Assert.Equal(new[] { true, true, true, true }, child1.GetGenes());
        Assert.Equal(new[] { false, false, false, false }, child2.GetGenes());
    }

    [Fact]
    public void Cross_DifferentLength_Throws()
    {
        var p1 = new Chromosome(new[] { true, false }, 0);
        var p2 = new Chromosome(new[] { true, false, true }, 0);
        var crossover = new SinglePointCrossover(1.0);

        Assert.Throws<ArgumentException>(() => crossover.Cross(p1, p2, FixedRng));
    }
}
