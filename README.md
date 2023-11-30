# InferredBugs

InferredBugs is a metadata-rich dataset of bugs and fixes in Java and C# programming languages extracted with the [Infer](https://fbinfer.com/) static analyzer tool. The dataset has been constructed by systematically analyzing open-source repositories, scrutinizing each commit to identify the inception and resolution of bugs.

This dataset served as the foundational corpus for training the InferFix model, as detailed in the ESEC/FSE paper titled *"InferFix: End-to-End Program Repair with LLMs"*.

This dataset encompasses comprehensive metadata for each identified bug, making it an invaluable resource for advancing research in the field of Automated Program Repair. Additionally, it facilitates large-scale empirical analyses on bugs, empowering researchers to glean insights and advance the understanding of software defects and their resolutions.

## Dataset Structure
The InferredBugs dataset is meticulously organized by programming language, with distinct folders for `java` and `csharp` located in the root directory. The structure of each language-specific folder is as follows:

- `<repository>`: Name of the repository subjected to analysis.
    - `<bug_id>`: Bug identifier for the given repository.
        - `bug.json`: Metadata associated with the bug, meticulously extracted by Infer. This file includes rich information such as `bug_type`, `severity`, and localization details.
        - `commit_info.json`: Metadata associated with the commit that rectified the bug. It encompasses essential information like `hash` and `message`.
        - `file_before.txt`: The content of the file before the bug fix.
        - `file_after.txt`: The content of the file after the bug fix.
        - `method_before.txt`: The content of the buggy method before the fix.
        - `method_after.txt`: The content of the method after the fix.

## Citation

```
@inproceedings{inferifx2023,
    title={InferFix: End-to-End Program Repair with LLMs},
    author={Matthew Jin, Syed Shahriar, Michele Tufano, Xin Shi, Shuai Lu, Neel Sundaresan, and Alexey Svyatkovskiy},
    booktitle={2023 31st ACM Joint European Software Engineering Conference and Symposium on the Foundations of Software Engineering (ESEC/FSE 2023)},
    year={2023},
    organization={ACM}
}
```


## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
