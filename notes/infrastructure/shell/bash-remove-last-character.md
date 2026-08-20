# bash 中如何去除變數最後一位的資料

1. 使用 `${var%pattern}` 或 `${var%%pattern}`：

    ```bash
    # 定義變數
    string="abcdefg"

    # 去除最後一位（對應到 "g"）
    echo "${string%?}"  # 輸出 "abcdef"

    # 去除最後一個匹配到的 "e"
    echo "${string%e*}"  # 輸出 "abcd"

    # 使用%%會去除最後一個匹配到的pattern，而不是最後一個
    echo "${string%%e*}"  # 輸出 "abcd"

    ```

2. 使用 `cut` 指令：

    ```bash
    # 定義變數
    string="abcdefg"

    # 使用 cut 指令切割字串，-c 參數指定要切割的位置
    echo "$(cut -c 1-$(( ${#string} - 1 )) <<< "$string")"  # 輸出 "abcdef"
    ```

3. 使用 `sed` 指令：

    ```bash
    # 定義變數
    string="abcdefg"

    # 使用 sed 指令替換字串
    echo "$(sed 's/.$//' <<< "$string")"  # 輸出 "abcdef"
    ```

4. 使用 `awk` 指令：

    ```bash
    # 定義變數
    string="abcdefg"

    # 使用 awk 指令處理字串
    echo "$(awk '{print substr($0, 1, length($0)-1)}' <<< "$string")"  # 輸出 "abcdef"
    ```

## 相關文件

[簡明 Linux Shell Script 入門教學](https://blog.techbridge.cc/2019/11/15/linux-shell-script-tutorial/)
