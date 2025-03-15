---
layout: post
title: 2D Map generation using noises
description: Today I will try to explain, how can you generate 2D tile map and what noises can be useful for this purpose.
date: 2024-05-15
categories: [2D, Map, Noises]
tags: [2d, map, noises, monogame, tilemap]
---

Today I will try to explain, how can you generate 2D tile map and what noises can be useful for this purpose.

### Noises
I found, that a lot of games uses noises to generate maps instead of getting only random numbers. I will show you Perlin and Simplex noise, and how I used it in my game.

### Perlin Noise
Perlin Noise is first noise I used for map generation. It was developed by Ken Perlin in 1983. I used it for generate terrain tiles, like stone, dirt and also small caves.

#### C# Implementation
I used Monogame framework for Vector2 structure and generating texture from noises.
```csharp
public static float[] Generate(int width, int height, Random random)
{
    CalculatePermutation(out var permutation, random);
    CalculateGradients(out var gradients, random);
    var data = new float[width * height];
    int octaves = 8;

    /// track min and max noise value. Used to normalize the result to the 0 to 1.0 range.
    var min = float.MaxValue;
    var max = float.MinValue;

    var frequency = 0.5f;
    var amplitude = 1f;

    for (var octave = 0; octave < octaves; octave++)
    {
        /// parallel loop - easy and fast.
        Parallel.For(0
            , width * height
            , (offset) =>
            {
                var i = offset % width;
                var j = offset / width;
                var noise = Noise(i * frequency * 1f / width, j * frequency * 1f / height, permutation, gradients);
                noise = data[j * width + i] += noise * amplitude;

                min = Math.Min(min, noise);
                max = Math.Max(max, noise);

            }
        );

        frequency *= 2;
        amplitude /= 2;
    }

    return data;
}

private static float Noise(float x, float y, int[] permutation, Vector2[] gradients)
{
    var cell = new Vector2((float)Math.Floor(x), (float)Math.Floor(y));
    var total = 0f;
    var corners = new[] { new Vector2(0, 0), new Vector2(0, 1), new Vector2(1, 0), new Vector2(1, 1) };

    foreach (var n in corners)
    {
        var ij = cell + n;
        var uv = new Vector2(x - ij.X, y - ij.Y);

        var index = permutation[(int)ij.X % permutation.Length];
        index = permutation[(index + (int)ij.Y) % permutation.Length];

        var grad = gradients[index % gradients.Length];

        total += Q(uv.X, uv.Y) * Vector2.Dot(grad, uv);
    }

    return Math.Max(Math.Min(total, 1f), -1f);
}

private static void CalculatePermutation(out int[] permutation, Random random)
{
    permutation = Enumerable.Range(0, 256).ToArray();

    /// shuffle the array
    for (var i = 0; i < permutation.Length; i++)
    {
        var source = random.Next(permutation.Length);
        (permutation[source], permutation[i]) = (permutation[i], permutation[source]);
    }
}

private static void CalculateGradients(out Vector2[] grad, Random random)
{
    grad = new Vector2[256];

    for (var i = 0; i < grad.Length; i++)
    {
        Vector2 gradient;

        do
        {
            gradient = new Vector2((float)(random.NextDouble() * 2 - 1), (float)(random.NextDouble() * 2 - 1));
        }
        while (gradient.LengthSquared() >= 1);

        gradient.Normalize();

        grad[i] = gradient;
    }
}

private static float Drop(float t)
{
    t = Math.Abs(t);
    return 1f - t * t * t * (t * (t * 6 - 15) + 10);
}

private static float Q(float u, float v)
{
    return Drop(u) * Drop(v);
}
```

#### Results
For higher frequency and less octaves, result is more "noisy".

| Octaves | Frequency | Result |
|---------|-----------|--------|
| 4       | 0.5       | ![Perlin 4](/img/2024-05-15-noises/perlin_4.png)      |
| 4       | 0.7       | ![Perlin 4](/img/2024-05-15-noises/perlin_4_1.png)      |
| 8       | 0.5       | ![Perlin 8](/img/2024-05-15-noises/perlin_8.png)      |
| 8       | 0.7       | ![Perlin 8](/img/2024-05-15-noises/perlin_8_1.png)      |
| 12      | 0.5       | ![Perlin 12](/img/2024-05-15-noises/perlin_12.png)      |
| 12      | 0.7       | ![Perlin 12](/img/2024-05-15-noises/perlin_12_1.png)      |

#### How to use in terrain generator?
For Perlin noise generator part I have noise result, list of object I want to be placed at this stage, and maxValue for ignoring value higher than it. With maxValue we can simply generate caves.
```csharp
float[] result = noise.Generate(Global.Map.Size.X, Global.Map.Size.Y, random);
List<MapObject> mapObjects = new (); //Fill with dirt object, stone etc.
float maxValue = 0.2f; //Range 0-1, Higher value means less caves
Global.Map.MapCells.ForEach((p, c) =>
{
    float val = result[p.Y * Global.Map.Size.X + p.X];

    if (val < maxValue)
    {
        float step = maxValue / (float)mapObjects.Count;
        for (int a = 0; a < mapObjects.Count; a++) //Can be done without for loop
        {
            if (val < (step * (float)a))
            {
                Global.Map.Place(mapObjects[a], p);
                break;
            }
        }
    }
});
```

### Simplex Noise
Simplex Noise is comparable to Perlin Noise, but with less complexity and higher directional artifacts. I used it for ore generation.

#### C# Implementation

```csharp
public static float[,] Generate(int width, int height, Random random)
{
    CalculatePermutation(out var permutation, random);
    float scale = 0.1f;

    var values = new float[width, height];
    for (var i = 0; i < width; i++)
        for (var j = 0; j < height; j++)
            values[i, j] = (Noise(i * scale, j * scale, permutation) * 128f + 128f) / 255f;

    return values;
}

private static float Noise(float x, float y, int[] permutation)
{
    const float F2 = 0.366025403f; // F2 = 0.5*(sqrt(3.0)-1.0)
    const float G2 = 0.211324865f; // G2 = (3.0-Math.sqrt(3.0))/6.0

    float n0, n1, n2; // Noise contributions from the three corners

    // Skew the input space to determine which simplex cell we're in
    var s = (x + y) * F2; // Hairy factor for 2D
    var xs = x + s;
    var ys = y + s;
    var i = FastFloor(xs);
    var j = FastFloor(ys);

    var t = (i + j) * G2;
    var X0 = i - t; // Unskew the cell origin back to (x,y) space
    var Y0 = j - t;
    var x0 = x - X0; // The x,y distances from the cell origin
    var y0 = y - Y0;

    // For the 2D case, the simplex shape is an equilateral triangle.
    // Determine which simplex we are in.
    int i1, j1; // Offsets for second (middle) corner of simplex in (i,j) coords
    if (x0 > y0) { i1 = 1; j1 = 0; } // lower triangle, XY order: (0,0)->(1,0)->(1,1)
    else { i1 = 0; j1 = 1; }      // upper triangle, YX order: (0,0)->(0,1)->(1,1)

    // A step of (1,0) in (i,j) means a step of (1-c,-c) in (x,y), and
    // a step of (0,1) in (i,j) means a step of (-c,1-c) in (x,y), where
    // c = (3-sqrt(3))/6

    var x1 = x0 - i1 + G2; // Offsets for middle corner in (x,y) unskewed coords
    var y1 = y0 - j1 + G2;
    var x2 = x0 - 1.0f + 2.0f * G2; // Offsets for last corner in (x,y) unskewed coords
    var y2 = y0 - 1.0f + 2.0f * G2;

    // Wrap the integer indices at 256, to avoid indexing perm[] out of bounds
    var ii = Mod(i, 256);
    var jj = Mod(j, 256);

    // Calculate the contribution from the three corners
    var t0 = 0.5f - x0 * x0 - y0 * y0;
    if (t0 < 0.0f) n0 = 0.0f;
    else
    {
        t0 *= t0;
        n0 = t0 * t0 * Grad(permutation[ii + permutation[jj]], x0, y0);
    }

    var t1 = 0.5f - x1 * x1 - y1 * y1;
    if (t1 < 0.0f) n1 = 0.0f;
    else
    {
        t1 *= t1;
        n1 = t1 * t1 * Grad(permutation[ii + i1 + permutation[jj + j1]], x1, y1);
    }

    var t2 = 0.5f - x2 * x2 - y2 * y2;
    if (t2 < 0.0f) n2 = 0.0f;
    else
    {
        t2 *= t2;
        n2 = t2 * t2 * Grad(permutation[ii + 1 + permutation[jj + 1]], x2, y2);
    }

    // Add contributions from each corner to get the final noise value.
    // The result is scaled to return values in the interval [-1,1].
    return 40.0f * (n0 + n1 + n2); // TODO: The scale factor is preliminary!
}

private static void CalculatePermutation(out int[] permutation, Random random)
{
    permutation = Enumerable.Range(0, 256).ToArray();
    permutation = permutation.Concat(permutation).ToArray();

    /// shuffle the array
    for (var i = 0; i < permutation.Length; i++)
    {
        var source = random.Next(permutation.Length);
        (permutation[source], permutation[i]) = (permutation[i], permutation[source]);
    }
}

private static int FastFloor(float x)
{
    return (x > 0) ? ((int)x) : (((int)x) - 1);
}

private static int Mod(int x, int m)
{
    var a = x % m;
    return a < 0 ? a + m : a;
}

private static float Grad(int hash, float x, float y)
{
    var h = hash & 7;      // Convert low 3 bits of hash code
    var u = h < 4 ? x : y;  // into 8 simple gradient directions,
    var v = h < 4 ? y : x;  // and compute the dot product with (x,y).
    return ((h & 1) != 0 ? -u : u) + ((h & 2) != 0 ? -2.0f * v : 2.0f * v);
}
```

#### Results
For ore generation in my game I used scale = 0.1f

| Scale | Result |
|-------|--------|
| 0.1   | ![Simplex 0.1](/img/2024-05-15-noises/simplex_0.1.png)      |
| 0.5   | ![Simplex 0.5](/img/2024-05-15-noises/simplex_0.5.png)      |
| 0.9   | ![Simplex 0.9](/img/2024-05-15-noises/simplex_0.9.png)      |

#### How to use?
For ore generation I'm filtering noise result and getting max value in specified area.
```csharp
public readonly struct FreqEntry
{
    public readonly int Power;
    public readonly Point Point;

    public FreqEntry(int power, Point point)
    {
        Power = power;
        Point = point;
    }
}

public static List<FreqEntry> HigherFrequences(float[,] result, int[] freq)
{
    List<FreqEntry> points = new List<FreqEntry>();

    result.ForEach((p, v) =>
    {
        for (int a = 0; a < freq.Length; a++)
        {
            Point maxPoint = GetMaxPoint(result, p, freq[a]);
            points.Add(new FreqEntry(freq[a], maxPoint));
        }
    });

    return points;
}

private static Point GetMaxPoint(float[,] array, Point p, int distance)
{
    float max = 0;
    Point maxPoint = Point.Zero;

    for (int x = -distance; x <= distance; x++)
    {
        for (int y = -distance; y <= distance; y++)
        {
            int xn = p.X + x;
            int yn = p.Y + y;

            if (array.Inside(new Point(xn, yn)))
            {
                float v1 = array[xn, yn];
                if (max < v1)
                {
                    max = v1;
                    maxPoint = new Point(xn, yn);
                }
            }
        }
    }

    return maxPoint;
}
```

```csharp
int[] freq = new int[] { 4, 6, 8, 11 };

//4 = Common
//6 = Normal
//8 = Rare
//11 = Very rare

var freqPoints = NoisePatterns.HigherFrequences(result, freq);

foreach (var freqPoint in freqPoints)
{
    var point = freqPoint.Point;
    var power = freqPoint.Power;

    List<MapObject> ent = new (); //Fill it with ores with rarity = power. In my project I stored it in Dictionary<int, List<MapObject>> where key is power, and this list is a value

    if (ent.Count == 0)
    {
        continue;
    }
    Global.Map.Place(ent.ToArray().Random(random), point);
}

public static T Random<T>(this T[] array, Random random)
{
    int index = random.Next(0, array.Length);
    if (index == array.Length)
    {
        index--;
    }

    return array[index];
}
```

### Summary
I hope this post is understandable. I'm trying my best to share my ideas with others :smile: 

### Sources
- <https://stackoverflow.com/questions/8659351/2d-perlin-noise>
- <https://github.com/WardBenjamin/SimplexNoise>
