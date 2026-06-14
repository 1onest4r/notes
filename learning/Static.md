unlike stack and heap it saves the variable onto the binary/object meaning it lives in the code rather than in memory?
it's cool because it just makes sure variable gets initialized only once
and gets remembered throughout the relative scope by getting remembered

example usecase:

```
#include <iostream>
using namespace std;

void foo() {
  static int s_var = 0;
  s_var++;

  cout << s_var << endl;
};

int main() {
  int t = 10;
  while(t--) {
    foo();
  }


  return 0;
}

```
