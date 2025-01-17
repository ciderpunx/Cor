Prev: [Classes](classes.md)  
Next: [Attributes](attributes.md)

---

# Corinna Class Construction

For object construction, we provide a list of needed steps, but then we'll
have pseudocode to make the construction process explicit.

Note: there are a number of references to MRO order, but we'll likely
be single inheritance for the MVP. Thus, MRO order is parent to child and
reverse MRO is child to parent.

Anything which can be removed from this will make object construction
faster. Anything which can be pushed to compile-time will make object
construction faster.

This is almost certainly incorrect, but it's a start.

Also, roles get an ADJUST phaser now

1. Check that the even-sized list of args to new() are not duplicated
   (stops the new( this => 1, this => 2 ) error)
2. And that the keys are not references
3. Walk through classes in reverse MRO order. Croak() if any attribute
   name is reused
4. After previous step, if we have any extra keys passed to new() which cannot be
   allocated to a slot, throw an exception
5. For the internal NEW phaser, assign all values to their correct slots in
   reverse mro order
6. Call all ADJUST phasers in reverse MRO order (no need to validate here because
   everything should be checked at this point)

```perl
# 1. Check that the even-sized list of args to new() are not duplicated
#   (stops the new( this => 1, this => 2 ) error)
my @args = ... get list passed to new()
  unless ( !( @args % 2 ) ) {
    croak("even-sized list required");
}

# 2. And that the keys are not references
my %arg_for;
while (@args) {
    my ( $key, $value ) = splice @args, 0, 2;
    if ( ref $key ) {
        croak("$key must not be a ref");
    }
    if ( exists $arg_for{$key} ) {
        croak("duplicate key $key detected");
    }
    $arg_for{$key} = $value;
}

# and almost certainly incorrect
my %orig_args = %arg_for;    # shallow copy
my %constructor_args;


# 3. Walk through classes in reverse MRO order. Croak() if any attribute
#    name is reused
my @duplicate_constructor_args;
foreach my $class (@reverse_mro) {
    my @roles = roles_from_class($class);
    foreach my $thing ( $class, @roles ) {
        foreach my $name ( get_slots_with_new_attribute($thing) ) {
            if ( my $other_class = $constructor_args{$name} ) {
                # XXX Warning! This may be a bad thing
                # If you don't happen to notice that some parent class has done
                # `has $cutoff :param = 42;`
                # then you might accidentally write:
                # `has $cutoff :param = DateTime->now->add(days => 7);`
                # instead, we probably need some way of signaling this to the
                # programmer. A compile-time error would be good.
                push @duplicate_constructor_args 
                  => "Arg $name in $thing already used in $other_class";
            }
            $constructor_args{$name} = $class;
        }
    }
}
if (my $error = join '  ' => @duplicate_constructor_args) {
    croak($error);
}


# 4. After previous step, if we have any extra keys passed to new() which cannot be
#    allocated to a slot, throw an exception
# this works because by the time we get to the final class, all keys
# should be accounted for. Stops the issue of Class->new(feild => 4) when
# the attribute is `has $field :param = 3;`
my @bad_keys;
foreach my $key ( keys %arg_for ) {
    push @bad_keys => $key unless exists $constructor_args{$key};
}
if (@bad_keys) {
    croak(...);
}

# phaser NEW
# 5. For the internal NEW phaser, assign all values to their correct slots in
#    reverse mro order
my @slot_values;
foreach my $this_class (@reverse_mro) {
    my @roles = roles_from_class($class);
    foreach my $thing ( $class, @roles ) {
        foreach my $slot_name ( get_slots_in_initialization_order($thing) ) {
            push @slot_values => $arg_for{$slot_name};
        }
    }
}
my $self = bless \@slot_values => $class;

# Call all ADJUST phasers
# 6. Call all ADJUST phasers in reverse MRO order (no need to validate here because
#    everything should be checked at this point)
foreach my $class (@reverse_mro) {
    my @roles = roles_from_class($class);
    foreach my $thing ( $class, @roles ) {
        $thing::ADJUST ( $self, %arg_for );    # phaser, not a method
    }
}

# MOP stuff

MOP::Class {
    method get_slots_with_new_attributes($class_or_role) {
        return
          grep { $self->has_attribute( 'new', $_ ) }
          get_all_slots($class_or_role);
    }

    method get_slots_in_initialization_order($class_or_role) {
        # get_all_slots($class_or_role) should return them in declaration order
        my @slots = get_all_slots($class_or_role);
        my @ordered;
        my $constructor_args_processed = 0;
        while (@slots) {
            my $slot = shift @slots;
            if ( $self->has_attribute( 'new', $slot ) ) {
                push @ordered => $slots;
                my @remaining;
                foreach my $slot (@slots) {
                    if ( $self->has_attribute( 'new', $slot ) ) {
                        push @ordered => $slot;
                    }
                    else {
                        push @remaining => $slot;
                    }
                }
                @slots = @remaining;
            }
            else {
                push @ordered => $slot;
            }
        }
    }
}
```

---

Prev: [Classes](classes.md)  
Next: [Attributes](attributes.md)
