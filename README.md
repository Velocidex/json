# regex library

This is a fork of the standard Golang Regex library addressing a major
limitation. The standard library allows a type of define a custom
marshaller, but this does not help when the user of the library is
trying to marshal types that they do not own, or they wish to
customize the marshalling specifically for each operation.

If you want to customize marshalling you typically need to satisfy the
Marshaller interface in your type.

This is OK for your own types but what if I have third party types?
Using the standard library encoder means that I have to wrap the
foreign type in a new type for every occurance of that type - this is
error prone and tedius.

## Example

The most common example of this shortfall occurs when serializing
time.Time objects. Although time.Time implements its own MarshalJSON()
method, by default it encodes it into RFC3339 using local time
offsets. However, most people want to always serialize into UTC offset
so it is easier to string compare JSON times.

Since it is actually impossible to force a Golang binary to change
it's localtime in a reliable way after initialization (see
https://groups.google.com/g/golang-nuts/c/d2AG5LNuKms/m/YM7zD8D9AwAJ)
we need a way to specify that time.Time should be serialized into a
particular timezone at the encoder level.

i.e. every encode operation needs to be able to specify context to the
encoder so the encoder can produce context aware output (e.g. this
user prefers their JSON in Z time, this one in AEST etc).

Since we can not actually change time.Time's MarshalJSON() method we
need a way to override it.

This library provides this way by specifying callback marshallers for
different types:

```
    m := .... some object containing possible time.Time

    location, _ = time.LoadLocation("Australia/Brisbane")
    cb := func(v interface{}) ([]byte, error) {
        switch t := v.(type) {
        case time.Time:
            // Marshal the time in the desired timezone.
            return t.In(location).MarshalJSON()
        }
        return nil, EncoderCallbackSkip
    }

    // Assign this callback to time.Time
    enc_opts := NewEncOpts().WithCallback(time.Time{}, cb)

    // Marshal with those options.
    b, err := MarshalWithOptions(m, enc_opts)
    ...
```

The options provide the marshalling context and allow to override (or
implement) type's MarshalJSON(). This dynamically applies to types,
even if they are out of tree.
