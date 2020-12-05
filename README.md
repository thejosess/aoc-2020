# aoc-2020

Advent of Code 2020

## To build and run the project

This project uses [cargo-aoc](https://github.com/gobanos/cargo-aoc). More detailed instructions can be found at that project's [README](https://github.com/gobanos/cargo-aoc/blob/master/README.md) file.

1. Create an account at adventofcode.com
2. Get the value for your session cookie and configure `cargo-aoc`:

```
cargo aoc credentials -s TOKEN
```

3. Build and run the code with:

```
cargo aoc
```

## Log

### Day 1

I've used a nested loop to iterate over the input, with the only optimization of not iterating over pairs of numbers that have been tried before (since addition is commutative):

```rust
for i in 0..entries.len() {
  for j in (i + 1)..entries.len() {
    // ...
  }
```

For **part 2** I added a third nested loop to get the third element to add. We avoid entering into this loop if we know the solution is impossible to reach using that combination (if the first two element already exceed the target).

```rust
for i in 0..entries.len() {
  for j in (i + 1)..entries.len() {
    if entries[i] + entries[j] > TARGET {
      continue;
    }

    for k in (j + 1)..entries.len() {
      // ...
    }
  }
}
```

### Day 2

I learned a bunch of Rust with today's puzzle:

- How to create regexes with named capture groups.
- Use the `lazy_static!` macro to compute a value only once
- How to return a closure

I used a regex to parse the policy and extract its config:

```rust
lazy_static! {
  static ref PARSER: Regex =
    Regex::new(r"(?P<num1>\d+)\-(?P<num2>\d+)\s(?P<character>\w)").unwrap();
}
```

And then I have two functions (one per password policy) that return a closure that performs the password validation. For instance, this is the closure returned for the first policy:

```rust
Box::new(move |text| {
  let amount = text
    .chars()
    .filter(|&x| x == policy.character)
    .collect::<String>()
    .len() as u32;
  amount >= policy.num1 && amount <= policy.num2
})
```

### Day 3

I took this as an opportunity to learn how to implement a custom iterator in Rust.

```rust
impl SlopeIterator {
  fn new(slope: (usize, usize), map: &Map) -> Self {
    SlopeIterator {
      slope: slope,
      current: (0, 0),
      limit: (usize::MAX, map.height - 1),
    }
  }
}

impl Iterator for SlopeIterator {
  type Item = (usize, usize);

  fn next(&mut self) -> Option<(usize, usize)> {
    if self.current.0 > self.limit.0 || self.current.1 > self.limit.1 {
      return None;
    }

    let res = self.current;
    self.current.0 += self.slope.0;
    self.current.1 += self.slope.1;

    return Some(res);
  }
}
```

With that iterator, traversing the map to look for cells with trees it's quite simple:

```rust
iterator.fold(0, |total, (x, y)| {
  let inc = match map.get_cell(x, y) {
    Some(Cell::Tree) => 1,
    _ => 0,
  };

  total + inc
})
```

I had problems with **part 2**, because it asks to multiply partial results and supply that result as the answer for the puzzle, but I was using `u32` and there was an overflow I wasn't noticing… I got the real result once I switched to `u64`.

### Day 4

Today's puzzles were simple, but tedious. I'm thankful for how easy is to add and run unit tests in Rust.

I decided to use "new" (for me in Rust!) data structures: `HashMap` to hold the passport information, and `HashSet` for the required fields. This way, it's easy to see which fields are missing:

```rust
let required_fields: HashSet<String> = ["ecl", "pid", "eyr", "hcl", "byr", "iyr", "cid", "hgt"]
    .iter()
    .map(|x| x.to_string())
    .collect();

let passport_fields: HashSet<String> = passport.keys().cloned().collect();
let missing: Vec<&String> = required_fields.difference(&passport_fields).collect();
```

For part 2, I ended up creating separate functions to validate each data type –so I could unit test them separately. Once that's in place, checking whether a field is valid with `match` is quite readable:

```rust
let is_valid: bool = match field.as_str() {
  "byr" => is_valid_number(value, 1920, 2002),
  "iyr" => is_valid_number(value, 2010, 2020),
  "eyr" => is_valid_number(value, 2020, 2030),
  "hgt" => is_valid_height(value),
  "hcl" => is_valid_hex_number(value),
  "ecl" => is_valid_eye_color(value),
  "pid" => is_valid_passport_id(value),
  "cid" => true,
  _ => false,
};
```

### Day 5

Today's been more fun that yesterday. I created a recursive function to do this partition of left/front vs right/back, and got to play with ranges in Rust:

```rust
pub fn locate(range: Range<usize>, locator: &[Dir]) -> usize {
  let length = range.len();

  if length > 2 {
    let new_range = match locator[0] {
      Dir::Lower => range.start..(range.start + length / 2),
      Dir::Higher => (range.start + length / 2)..range.end,
    };
    locate(new_range, &locator[1..])
  } else {
    match locator[0] {
      Dir::Lower => range.start,
      Dir::Higher => range.end - 1, // -1 because range.end is not inclusive
    }
  }
}
```

For the second part of the puzzle, the hardest bit was to understand the wording of the problem 😅
