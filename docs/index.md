```cpp title="welcome"
#include <iostream>
#include <string>
#include <vector>
#include <map>

class PersonalProfile {
private:
    std::string name;
    std::string occupation;
    std::vector<std::string> interests;
    std::map<std::string, std::string> contact;

public:
    PersonalProfile() {
        name = "peiyuzou";
        occupation = "game programmer";
        interests = {
            "game", "game dev", "coffee",
            "hotpot", "bbq", "Anything fun"
        };
        contact["GitHub"] = "https://github.com/peiyuzou";
        contact["Email"] = "18280318713@email.com";
    }
};

int main() {
    std::cout << "Thanks for visit, hope it's helpful to you!" << std::endl;
    return 0;
}

```