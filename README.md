# 1. 目的
利用爬虫抓取信息：今年只招收了一名学生的导师的教师主页中的研究方向。
# 2. 准备
1. 准备一个excel表单，含有每个学生对应的导师
2. 安装chrome driver和selenium （准备 Chrome 浏览器）
参考：https://blog.csdn.net/zywmac/article/details/137470787
# 3. 思路
1. 先在excel表单中用公式筛选出符合条件的导师姓名
2. 使用selenium模拟浏览器动作搜索每个导师的教师主页，抓取”研究领域“的文本内容
# 4. 过程
## 1. excel中使用公式进行预处理
1. 假设导师姓名都在A列，从 A2 开始。在 B2 单元格中输入以下公式，然后向下拖动填充公式到其他单元格：
` =COUNTIF(A:A, A2)`
计算 A 列中每个名字出现的次数。
2. 提取只出现一次的名字,在 C2 单元格中输入以下公式：
`=IF(B2=1, A2, "")`
然后向下填充公式，显示出现次数为 1 的名字。（该列改名为”导一“）
## 2. python环境内操作爬虫
1. 准备数据
```python
df = pd.read_excel("硕士对应导师信息表.xlsx")
df = df.dropna()["导一"] 
```
2. 配置 Selenium WebDriver (确保你已经下载并配置好 ChromeDriver 或其他浏览器驱动)
```python
driver = webdriver.Chrome()  # 如果你用的是 Chrome 浏览器
driver.get("https://www.bing.com")

# 定义一个空列表，用来存储搜索结果地址
results = []

time.sleep(2)  # 等待页面加载
```
3. 模拟浏览器动作搜索导师信息，以哈尔滨工业大学的导师为例，搜索关键词条"导师名字-哈尔滨工业大学教师个人主页 - HIT"
```python
# 遍历每个名字并进行搜索
for name in df['导一']:
    search_query = f"{name} - 哈尔滨工业大学教师个人主页 - HIT"
    
    # 找到 Bing 搜索框并输入查询内容
    search_box = driver.find_element(By.NAME, "q")
    search_box.clear()
    search_box.send_keys(search_query)
    search_box.send_keys(Keys.RETURN)
    
    time.sleep(2)  # 等待页面加载
    
    # 获取第一个搜索结果的地址
    try:
        first_result = driver.find_element(By.CSS_SELECTOR, "li.b_algo h2 a")
        first_result_url = first_result.get_attribute("href")
    except Exception as e:
        first_result_url = "未找到结果"
    
    results.append(first_result_url)
    time.sleep(1)  # 等待页面加载

# 将结果添加到 DataFrame 新的一列
df['url'] = results

# 关闭浏览器
driver.quit()

# 显示带有新列的 DataFrame
print(df)
```
4. 随便进入一个导师的教师主页，右键查看HTML源码，观察"研究方向"在哪个元素内。以哈尔滨工业大学的导师为例，一般导师的研究领域在第二个div class="part_box"中的div class="editor_content"中。
观察可知，这些div被伪元素::before ::after包裹，为了消除其对获取文本的影响，可以使用
```python
content = driver.execute_script("return window.getComputedStyle(arguments[0], '::before').getPropertyValue('content') + arguments[0].textContent + window.getComputedStyle(arguments[0], '::after').getPropertyValue('content');", div_element)

```

# 5. 遇到的问题
1. ValueError: Timeout value connect was ……, but it must be an int, float or None
解决方法：selenium库和urllib3库版本不兼容，将urllib3版本降低。
参考：https://blog.csdn.net/weixin_60535956/article/details/131660133
2. ChromeDriver与chrome浏览器不兼容
解决方法：更新ChromeDriver
参考：https://blog.csdn.net/IT_lw/article/details/121658468
3. 不同导师的主页源码中，相同的板块class不同，内容顺序不同，有些缺乏有效信息，给爬取工作带来极大的挑战性。我无法解决这些问题，所以选择无视这些缺失数据。Anyway，我很想知道哈尔滨工业大学这个教师主页网站是谁开发维护的。
# 6. 参考
1. https://blog.csdn.net/zywmac/article/details/137470787
2. https://blog.csdn.net/weixin_60535956/article/details/131660133
