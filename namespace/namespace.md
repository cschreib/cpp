I am wondering if it would be possible to extend the notion of "name space" to C++ constructs which are not stricly speaking C++ namespaces (as in "namespace std {}"), but that do carry a name space with them: classes, structs and enums.
It seems strange that you cannot manipulate these name spaces like you could for regular namespaces.
The idea would be to allow importing the scope of classes, structs and enums using the "using namespace" syntax, as well as enabling ADL (I do not know if the latter would be a mistake or if it would actually allow new interesting constructs).


Hence:
```c++
#include <iostream>

struct some_struct {
    int m_i;
    void foo();

    static const int pi = 3;
    static void bar();
    using data_type = double;
};

int main() {
    // Import the names from 'some_struct' into the current namespace
    using namespace some_struct;

    // Now we can use the types declared inside 'some_struct'
    data_type my_pi = 3.14159;
    // ... the static members (variables and functions)
    std::cout << pi << std::endl;
    bar();

    // ... but not non-static members
    m_i = 12; // error: missing a 'some_struct' object
    foo(); // error: missing a 'some_struct' object

    // Although the following could work, I suppose it should be forbidden
    // for the sake of consistency
    some_struct::int* p_m_i = &m_i; // error: do you mean 'some_struct::m_i'?

    return 0;
}
```

Apart from conceptual consistency, why would it actually be useful?

# Case 1: Simplify usage of enumerations in switch.
Note that it would also make the implementation of the following proposal almost trivial:
https://groups.google.com/a/isocpp.org/d/topic/std-proposals/f-P2TENSAaU/discussion

```c++
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

// Without this proposal
void treat_packet(packet::connection_failed p) {
    switch (p.reason) {
        using case_t = decltype(p.reason);
        case case_t::too_many_clients :
            std::cout << "server is crowded" << std::endl; break;
        case case_t::unexpected_packet :
            std::cout << "server talks nonsense" << std::endl; break;
        case case_t::internet_is_dead :
            std::cout << "please plug in your ethernet cable" << std::endl; break;
        // ...
    }
}

// With this proposal
void treat_packet(packet::connection_failed p) {
    switch (p.reason) {
        using namespace decltype(p.reason);
        case too_many_clients :
            std::cout << "server is crowded" << std::endl; break;
        case unexpected_packet :
            std::cout << "server talks nonsense" << std::endl; break;
        case internet_is_dead :
            std::cout << "please plug in your ethernet cable" << std::endl; break;
        // ...
    }
}
```

# Case 2: Provide better name import mechanisms for wrapper types.
```c++
// Without this proposal
template<typename T>
class vector_wrapper {
    std::vector<T> vec;

    using impl = std::vector<T>;

public :
    using impl::iterator;
    using impl::const_iterator;
    using impl::reverse_iterator;
    using impl::const_reverse_iterator;
    using impl::value_type;
    // etc...
};

// With this proposal
template<typename T>
class vector_wrapper {
    std::vector<T> vec;

public :
    using namespace std::vector<T>;
    // Imports all types declared in std::vector<T>
};
```

# Case 3: Allow templatized namespaces
```c++
namespace file_structure {
    struct v1 {
        struct world {
            uint32_t width, height;
            uint32_t num_region;

            friend std::istream& operator >> (std::istream&, world&);
        };

        struct region {
            int32_t x, y;
            bool visited;

            friend std::istream& operator >> (std::istream&, region&);
        };
    };

    // New version of the binary interface
    struct v2 {
        // World file did not change
        using world = v1::world;

        // We added a new datum to regions
        struct region {
            int32_t x, y;
            bool visited;
            bool owned;

            friend std::istream& operator >> (std::istream&, region&);
        };
    };
}

// Without this proposal
template<typename Version>
auto read_file(std::string filename) {
    std::ifstream file(filename);

    Version::world f;
    file >> f;

    std::vector<Version::region> regions(f.num_region);

    for (uint32_t i : std::range(f.num_region)) {
        file >> regions[i];
    }

    // Could load many other things

    return regions;
}

// With this proposal
template<typename Version>
auto read_file(std::string filename) {
    std::ifstream file(filename);

    using namespace Version;
    world f;
    file >> f;

    std::vector<region> regions(f.num_region);

    for (uint32_t i : std::range(f.num_region)) {
        file >> regions[i];
    }

    // Could load many other things

    return regions;
}

int main() {
    // Read saved data with the v1 format
    auto regsv1 = read_file<file_structure::v1>("save_v1.dat");
    // Or with the v2 format
    auto regsv2 = read_file<file_structure::v2>("save_v2.dat");

    return 0;
}
```

That's all I can think of right now, but I'm convinced others can end up with more interesting potential applications. Although cases 1 and 3 are cosmetic (we can do the same without the proposal, just with more verbose code), case 2 actually brings something new, as specifying all the imported types by hand will not allow automatic importing if std::vector<T> is modified later on.

If you have other ideas of what could be done with such a feature, or if you see dangerous side effects, please share.
