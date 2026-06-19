Configuration settings for all operations are in the `voices.toml` file. 
Configuration sections are hierarchical, so the settings in the `[voices.en.male]` section 
(for example) inherit values from the `[voices.en]` section, which in turn inherits values 
from the `[voices]` section. 

The `[azure]` section contains settings specific to Azure speech services.

# Tools

Use the following command when rendering text-to-speech:

```bash
dnx Microsoft.CognitiveServices.Speech.CLI
```

Use the az CLI to access the Azure speech service keys and resources.

Temporary files should be generated in the `temp` directory and cleaned up after use.