Considering the following situation:

    namespace packet {
        struct connection_failed {
            enum class reason_t {
                too_many_clients,
                unexpected_packet,
                internet_is_dead,
                // and so on and so forth...
            } reason;
        };
    }

    void treat_packet(packet::connection_failed p) {
        switch (p.reason) {
            // Handle all possible cases and react accordingly
            case packet::connection_failed::reason_t::too_many_clients :
                std::cout << "server is crowded" << std::endl; break;
            case packet::connection_failed::reason_t::unexpected_packet :
                std::cout << "server talks nonsense" << std::endl; break;
            case packet::connection_failed::reason_t::internet_is_dead :
                std::cout << "please plug in your ethernet cable" << std::endl; break;
            // ...
        }
    }

As you can see, the 'case' labels of the 'switch' statement are quite verbose and involve many repetitions that appear redundant. In this simple example, one can get away with it by creating an alias for the enumeration type, in which case the switch becomes more readable:

    using reason_t = packet::connection_failed::reason_t;
    switch (p.reason) {
        // Handle all possible cases and react accordingly
        case reason_t::too_many_clients :
            std::cout << "server is crowded" << std::endl; break;
        case reason_t::unexpected_packet :
            std::cout << "server talks nonsense" << std::endl; break;
        case reason_t::internet_is_dead :
            std::cout << "please plug in your ethernet cable" << std::endl; break;
        // ...
    }

This however comes with the creation of a new local alias 'reason_t' that outlives the switch itself. Although it does not matter for this simple function, it might be more troublesome in real world code bases where the switch is actually just a small fraction of the whole function.

I was thus wondering if it would be possible somehow to import the namespace of 'expr' in the 'switch (expr)' statement so that it becomes available for the 'case' labels only. You could call that a "Switch Argument Dependent Lookup" if you like (SADL sounds nice actually). The above code could then be written :

    switch (p.reason) {
        // Handle all possible cases and react accordingly
        case too_many_clients :
            std::cout << "server is crowded" << std::endl; break;
        case unexpected_packet :
            std::cout << "server talks nonsense" << std::endl; break;
        case internet_is_dead :
            std::cout << "please plug in your ethernet cable" << std::endl; break;
        // ...
    }

Now I realize this change would break existing code:

    int too_many_clients = 5;
    switch (p.reason) {
        case too_many_clients :
            // Although not very wise, this currently compiles and uses ::too_many_clients.
            // But this proposal brings another candidate,
            // namely packet::connection_failed::reason_t::too_man_clients,
            // so the situation is now ambiguous.
    }

There are several ways to avoid it.

1) Use SADL only if no candidate within the current namespace is found.
2) Introduce a new switch, e.g. 'switch enum' or 'enum switch', where SADL is used systematically. Such a name would imply that the switch expression must be of enum type (in addition, case labels could be forced to only involve enumerators of this enumeration, in order to increase type safety and add more value to the construct). If another name is found that does not involve 'enum', this constraint could be removed, but I feel that using this new switch explicitly asks for SADL, and therefore it would not make much sense to use such it on a plain integer.

Both solutions allow C++ code that would be otherwise invalid, and thus would not change the behavior of current code (except when SFINAE is involved in case of solution 1). I like the first solution more since it is more generic as it is not tied to enumerations in particular, and it is simple to learn as it does not involve any new construct.
