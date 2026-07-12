+++
title = "In praise of exhaustive destructuring"
date = "2026-07-12"
description = "Why you should always use exhaustive destructuring"
draft = false
# taxonomies.tags = [
#     "rust",
# ]
+++

When I learned Rust, I was frustrated by the fact that one had to list all fields or use the `..` syntax when destructuring a struct.

To illustrate this and because I am writing this during a heatwave, let's suppose we have the following struct:
```rust
struct WeatherReading {
    station_id: String,
    recorded_at: DateTime<Utc>,
    temperature: f64,
    humidity: f64,
    pressure: f64,
}
```
> **Note**: Except for `recorded_at`, all fields should be [newtypes](https://doc.rust-lang.org/rust-by-example/generics/new_types.html). Not done here for brevity.

Now I just want to retrieve `station_id` and `recorded_at`, why should I need to write:
```rust
let WeatherReading {
    station_id,
    recorded_at,
    ..
} = weather_reading;
```

In TypeScript, I could just do:
```ts
const { station_id, recorded_at } = weather_reading;
```

And in Haskell:
```hs
-- With NamedFieldPuns extension
let WeatherReading {
    station_id,
    recorded_at,
} = weather_reading

-- Or with RecordWildCards
let WeatherReading {..} = weather_reading
```

Not that much of a hassle, but that made me favor the `weather_reading.station_id`/`weather_reading.recorded_at` syntax.

However, in the past few months, I have started to greatly favor struct destructuring over accessing fields with the `.` syntax because that makes maintaining the software safer.

For example, we want to detect dangerous weather:
```rust
fn is_dangerous(weather_reading: &WeatherReading) -> bool {
    if weather_reading.temperature > 40.0 { // assuming °C, that's 104°F or 313.15K
        return true;
    }
    if weather_reading.humidity < 5.0 { // assuming percentage
        return true;
    }
    if weather_reading.pressure < 960.0 { // assuming hPa, that's 28.35 inHg
        return true;
    }
    return false;
}
```

That function is dangerous! To see why, let's suppose some stations now have an anemometer and are able to record wind speed. So, we add a new `wind_speed` field to the `WeatherReading` struct:

```rust
struct WeatherReading {
    station_id: String,
    recorded_at: DateTime<Utc>,
    temperature: f64,
    humidity: f64,
    pressure: f64,
    wind_speed: Option<f64>, // Again, this should be a newtype
}
```

All good! Let's ship!

Uh oh. Did you remember to update the `is_dangerous` function? Surely heavy wind should be reported as dangerous. Unfortunately, the Rust compiler did not warn us about it.

The good news is that we could have been warned if we wrote the function like this:

```rust
fn is_dangerous(
    WeatherReading {
        station_id: _,
        recorded_at: _,
        temperature,
        humidity,
        pressure,
    }: &WeatherReading
) -> bool {
    if *temperature > 40.0 { // assuming °C, that's 104°F or 313.15K
        return true;
    }
    if *humidity < 5.0 { // assuming percentage
        return true;
    }
    if *pressure < 960.0 { // assuming hPa, that's 28.35 inHg
        return true;
    }
    return false;
}
```

We would now get `error[E0027]: pattern does not mention field 'wind_speed'`.

Note that you must be explicit about unused fields and resist the temptation of using the `..` pattern for this trick to work.

Because `self` can't be destructured from the arguments list in Rust, if `is_dangerous` was written as a method, a bit of ceremony is necessary:

```rust
impl WeatherReading {
    fn is_dangerous(&self) -> bool {
        let WeatherReading {
            station_id: _,
            recorded_at: _,
            temperature,
            humidity,
            pressure,
        } = self;
        if *temperature > 40.0 { // assuming °C, that's 104°F or 313.15K
            return true;
        }
        if *humidity < 5.0 { // assuming percentage
            return true;
        }
        if *pressure < 960.0 { // assuming hPa, that's 28.35 inHg
            return true;
        }
        return false;
    }
}
```

I find this trick especially useful when writing `From` implementations between the different layers (data access, business logic, API) of a CRUD web service.
If you add a field in one layer, the compiler will force you to decide whether that field should be propagated to the other layers or not.

Another advantage is that if you see the same bunch of fields always destructured together while the others are ignored, that can be a good indication that maybe those fields should be extracted into their own struct.


If you are using TypeScript, there is [a trick using the `Required` type](https://medium.com/@_achou/using-TypeScript-to-statically-check-exhaustive-destructuring-7d56a7514127).

For Haskell, there is surprisingly no solution at the moment, though there is [a proposal](https://github.com/ghc-proposals/ghc-proposals/pull/436).
