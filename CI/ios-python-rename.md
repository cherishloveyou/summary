### 一些常用脚本



重命名代码

```python
# coding: utf-8
import os
import re
os.system('pip3 install chardet')
import chardet

ignore_list = [
    'make.width.mas_equalTo(labelSize.',
    'make.height.mas_equalTo([GCTTravelCardBottomView defaultSize].',
    'make.height.mas_equalTo(weakSelf.',
    'make.width.mas_equalTo(SC_FSW4_7(size.',
    'make.height.mas_equalTo(SC_FSW4_7(size.',
    'make.height.mas_equalTo(self.bootomViewSize.',
    'make.height.mas_equalTo([CGTTSPrirceDetailView sizeWithPriceItems:priceItems discountItems:discountsItems].',
    'make.width.mas_equalTo([UIScreen mainScreen].bounds.size.',
    'make.height.mas_equalTo(self.view.frame.size.height-self.customNavBar.frame.size.',
    'make.height.mas_equalTo([GCTTravelCardStatusView defaultSize].',
    'make.width.equalTo(self.promotionViewSize.',
    'make.width.mas_equalTo(rect.size.',
    'make.height.mas_equalTo(image.size.height*self.view.frame.size.width/image.size.',
    'make.height.mas_equalTo(self.view.frame.size.',
    'make.width.equalTo(weakSelf.rectInWindow.size.',
    'make.height.equalTo(weakSelf.rectInWindow.size.',
    'make.height.mas_equalTo([GCTTraveCardTrainView defaultSize].',
    'make.height.equalTo(self.promotionViewSize.',
    'make.height.mas_equalTo(rect.size.',
    'make.height.equalTo(weakSelf.rectInWindow.size.'
]


def check_ignore(tem_str):
    for value in ignore_list:
        if value in tem_str:
            return True
    return False


def splicing_str_list(tem_list):
    t_3 = ''
    for value in tem_list:
        if not t_3:
            t_3 = value
        else:
            t_3 = t_3 + '.' + value
    return t_3


def start():
    top_dir = input("输入项目路径或直接拖入文件夹（例:E:\svn_reposition\ZheJiang）：\n")
    if top_dir == '':
        print("未输入,程序结束")
        return
    else:
        top_dir = top_dir.strip()
    if top_dir == '':
        print("未输入,程序结束")
        return
    if not os.path.exists(top_dir):
        print("文件夹不存在")
        return
    if not os.path.isdir(top_dir):
        print("文件不存在")
        return
    mas_pattern = re.compile(r'.*([.]|[mas_])equalTo(.*)[.].*')
    for dir_path, sub_paths, files in os.walk(top_dir, False):
        for s_file in files:
            file_name, file_type = os.path.splitext(s_file)
            file_path = os.path.join(dir_path, s_file)
            if not file_type == '.h' and not file_type == '.m' and not file_type == '.mm':
                continue
            f = open(file_path, 'rb')
            data = f.read()
            encode_type = chardet.detect(data)["encoding"]
            f.close()
            new_content = []
            with open(file_path, mode='r', encoding=encode_type, errors='ignore') as file_object:
                file_content = file_object.readlines()
                for line in file_content:
                    if check_ignore(line):
                        new_content.append(line)
                        continue
                    if 'equal' not in line:
                        new_content.append(line)
                        continue
                    m = mas_pattern.match(line)
                    if not m:
                        new_content.append(line)
                        continue
                    g_list3 = m.group()
                    if not isinstance(g_list3, str):
                        new_content.append(line)
                        continue
                    t_1 = line.split('equalTo')
                    t_2 = t_1[1]
                    t_4 = t_2.split(')', maxsplit=1)
                    t_3 = t_4[0]
                    if t_3.endswith('.left'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('left', 'mas_left')
                        t_3 = splicing_str_list(tem_list_3)
                    elif t_3.endswith('.right'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('right', 'mas_right')
                        t_3 = splicing_str_list(tem_list_3)
                    elif t_3.endswith('.top'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('top', 'mas_top')
                        t_3 = splicing_str_list(tem_list_3)
                    elif t_3.endswith('.bottom'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('bottom', 'mas_bottom')
                        t_3 = splicing_str_list(tem_list_3)
                    elif t_3.endswith('.width'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('width', 'mas_width')
                        t_3 = splicing_str_list(tem_list_3)
                    elif t_3.endswith('.height'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('height', 'mas_height')
                        t_3 = splicing_str_list(tem_list_3)
                    elif t_3.endswith('.centerX'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('centerX', 'mas_centerX')
                        t_3 = splicing_str_list(tem_list_3)
                    elif t_3.endswith('.centerY'):
                        tem_list_3 = t_3.split('.')
                        tem_list_3[-1] = tem_list_3[-1].replace('centerY', 'mas_centerY')
                        t_3 = splicing_str_list(tem_list_3)
                    else:
                        new_content.append(line)
                        continue
                    new_line = ''
                    if len(t_1) == 2:
                        new_line = t_1[0] + 'equalTo' + t_3 + ')' + t_4[-1]
                    elif len(t_1) == 3:
                        new_line = t_1[0] + 'equalTo' + t_3 + ')' + t_4[-1] + 'equalTo' + t_1[-1]
                    else:
                        new_content.append(line)
                        continue
                    new_content.append(new_line)
            with open(file_path, mode='w', encoding=encode_type, errors='ignore') as file_object:
                file_object.writelines(new_content)
            print('处理完成: %s' % file_path)


start()
print('全部文件处理完成！')


```

