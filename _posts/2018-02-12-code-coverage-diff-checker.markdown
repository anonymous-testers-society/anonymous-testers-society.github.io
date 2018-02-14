---
layout: post
title: "Code coverage diff checker своими руками (на коленке)"
date: 2018-02-06 19:16:01 +0300
categories: tools
---

## Общие слова

Цель: ускоряем код-ревью, подсвечиваем непокрытые области кода в конкретной задаче как можно быстрее. делаем это автоматически, пишем в трекер.

Что на входе:
уже есть процесс CI, в команде уже используется код-ревью, потому что как же иначе?

Что на выходе:
робот пишет в задачу в трекере всё ли покрыто тестами как ожидалось

Как делаем:
берем diff кода, берем автоматический отчет по покрытию и сравниваем

## Пример реализации

уже используются в команде:
* git
* coverage (совершенно неважно какой инструмент, от него нужно чтобы он правильно считал покрытие и выдавал отчет в cobertura-like xml)
* python
* CI (Jenkins)
* Jira (для автоматизации генерим результат в json)

git diff выглядит так
```diff
$ git diff --unified=0
diff --git a/viewdata/viewbuilder/user.go b/viewdata/viewbuilder/user.go
index c6e6658b4..8797d9e93 100644
--- a/viewdata/viewbuilder/user.go
+++ b/viewdata/viewbuilder/user.go
@@ -16,7 +18,7 @@ func (handler *MainIndexHandler) Handle() error {
- meta := viewbuilder.NewMeta(block)
+
+ pm := pagemanager.FromContext(handler.Scope.AsContext())
+
+ pageManagerData := handler.getPageManagerData()
```

отчет о покрытиии (xml) выглядит так
```xml
...
<class name="Helper" filename="../catalog/catalog_helper.go" line-rate="0" branch-rate="0" complexity="0">
...
<lines>
<line number="100" hits="1"></line>
<line number="101" hits="1"></line>
<line number="101" hits="0"></line>
</lines>
...
```

сам скрипт может выглядеть, например, так (check_coverage.py)

```python
import os
import re
import json
import argparse
from subprocess import call

from lxml import etree

SKIPPED = ['_test', 'bindata_assetfs']
SOURCE_EXT = '.go'

def main():
    changes = []
    lines = dict()
    make_diff(changes, lines)
    if not lines:
        result_json = json.dumps({'TESTED': 'N/A', 'UNTESTED': 'N/A'})
        with open('./check_result.json', 'w') as f:
            f.write(result_json)
        exit('NOTHING TO CHECK IF COVERED')
    '''collect coverage reports'''
    parser = argparse.ArgumentParser()
    parser.add_argument('--reportsdir', action='store', type=str, help='coverage report directory')
    args = vars(parser.parse_args())
    if args['reportsdir']:
        reportsdir = args['reportsdir']
    else:
        print('Coverage report directory has to be provided!')
        exit(1)
    coverage_reports = collect_coverage_reports(reportsdir)
    check_results = dict()
    # Initially all lines are out of coverage
    untested_lines = dict()
    tested_lines = dict()
    for file in changes:
        check_results[file[0], file[1]] = -1

    '''parse coverage_reports'''
    parse_results(changes, check_results, coverage_reports, reportsdir)
    tested_lines_count = 0
    untested_lines_count = 0
    out_of_coverage_lines_count = 0
    for check_result in check_results:
        check_status = check_results[check_result]
        if check_status == 0:
            untested_lines_count += 1
            untested_lines[check_result] = lines[check_result]
        elif check_status == 1:
            tested_lines_count += 1
            tested_lines[check_result] = lines[check_result]
        else:
            out_of_coverage_lines_count += 1
    in_coverage_lines_count = tested_lines_count + untested_lines_count
    tested_lines_share = 0
    untested_lines_share = 0
    if in_coverage_lines_count > 0:
        tested_lines_share = float(tested_lines_count) / in_coverage_lines_count
        untested_lines_share = float(untested_lines_count) / in_coverage_lines_count
    if tested_lines_share == untested_lines_share == 0:
        result_json = json.dumps({'TESTED': 'N/A',
                                  'UNTESTED': 'N/A'})
    else:
        result_json = json.dumps({'TESTED': '{:.2%}'.format(tested_lines_share),
                                  'UNTESTED': '{:.2%}'.format(untested_lines_share)})
    with open('./check_result.json', 'w') as f:
        f.write(result_json)
    with open('./untested_lines.txt', 'w') as x:
        for line in sorted(untested_lines.keys()):
            x.write('{}:{} -> {}'.format(line[0], line[1], lines[line]))
    with open('./tested_lines.txt', 'w') as x:
        for line in sorted(tested_lines.keys()):
            x.write('{}:{} -> {}'.format(line[0], line[1], lines[line]))


def parse_results(changes, check_results, coverage_reports, reportdir):
    for file in changes:
        file_name = file[0]
        line_number = file[1]
        for coverage_report in coverage_reports:
            tree = etree.parse(coverage_report)
            for package in tree.getroot().find('packages'):
                for classes in package:
                    for class_node in classes:
                        if file_name in class_node.get('filename'):
                            for class_line in class_node[1]:
                                if str(line_number) == class_line.get('number'):
                                    if class_line.get('hits') == '0':
                                        if not check_results[file_name, line_number] == 1:
                                            check_results[file_name, line_number] = 0
                                    else:
                                        check_results[file_name, line_number] = 1


def collect_coverage_reports(reportdir):
    coverage_reports = [os.path.join(dp, f) for dp, dn, filenames in os.walk(reportdir) for f in filenames if
                        'coverage' in f and f.endswith('.xml')]
    if coverage_reports:
        return coverage_reports
    else:
        print('There is no any coverage report files according to coverage*.xml pattern')
        exit(1)


def make_diff(changes, lines):    
    with open("diff", "w") as out:
        call('git diff master --unified=0', shell=True, stdout=out)
    line_number = 0
    ignore_next = False
    with open('diff') as f:
        for line in f:
            '''
            ---
            -
            <any text>
            '''
            if line.startswith('-') or re.match('\w+', line):
                continue
            # blank line
            if re.match('\+\s*\n', line):
                line_number += 1
                continue
            # closing parenthesis
            if re.match('\+\s*\),*', line) or re.match('\+\s*}\s*(else)?\s*\{*', line):
                line_number += 1
                ignore_next = False
                continue
            # anonymous func defining and calling
            if re.match('\+\s*\w+\s*:=\s*func\(.*\).+\{\s*', line):
                line_number += 1
                continue
            # 1-line variables/const/etc
            if re.match('\+\s*var .+\n', line) \
                    or re.match('\+\s*struct .+\n', line) \
                    or re.match('\+\s*const .+\n', line) \
                    or re.match('\+\s*import .+\n', line) \
                    or re.match('\+\s*package .+\n', line):
                line_number += 1
            # commented line
            if re.match('\+\s*//.+', line):
                line_number += 1
                continue
            # Ignore func params if they are placed on separate line(s)
            if re.match('\+\s*func \w+\(\s*', line):
                ignore_next = True
                continue
            # +++
            if line.startswith('+++'):
                if re.match('.+/dev/null', line):
                    file_name = '/dev/null'
                    continue
                file_name = re.findall('\+{3} b/(.+)', line)[0]
                continue
		    # only source files excluding unittest
		    if not SOURCE_EXT in file_name:
		    	continue
		    if any(name in file_name for name in SKIPPED):
		    	continue
            # get filename
            if line.startswith('@@'):
                # non-func blocks (ignore all inside parenthesis except func calling)
                if re.match('.+var.+|.+const.+|.+import.+|.+package.+|.+struct.+', line):
                    ignore_next = True
                    continue
                else:
                    ignore_next = False
                line_number = int(re.findall('@@ .+ \+(\d+)', line)[0])
            if ignore_next:
                if not re.match('\+\s*\w+\s*=\s*.*\(.*\)', line):
                    line_number += 1
                    continue

            if re.match('\+.+', line):
                changes.append([file_name, line_number])
                lines[file_name, line_number] = line
                line_number += 1


main()
```

Как запустить, например, в докере:

Пример dockerfile
```
FROM jenkinsci/slave:3.7-1
USER root

# install some python staff
RUN yum install -y libxslt-devel python-devel python-pip git
RUN pip install lxml

USER jenkins
ENTRYPOINT ["jenkins-slave.sh"]
```

запуск из командной строки
```bash
git stash
git checkout master
git checkout <branch>
python check_coverage.py --reportsdir .
```

сам запуск при использовании groovy в Jenkins
```
def check_coverage(String project_dir) {
  try {
  	dir("${WORKSPACE}/src/coverage_checker") {
  	  git(branch: 'master', url: '<>')
  }
  dir(project_dir) {
  	sh returnStatus: true, script: """cp ${WORKSPACE}/src/coverage_checker/check_coverage.py ${WORKSPACE}/${project_dir}/check_coverage.py
	  	cd ${WORKSPACE}/${project_dir}/
	  	git stash
	  	git checkout master
	  	git checkout ${Branch_name}
	  	python check_coverage.py --reportsdir .
  	"""
  	}
  } catch (exception) {
  	print(exception)
  }
}

```

сохраняет файл в формате json с именем check_result.json

```json
{"TESTED": "13.33%", "UNTESTED": "86.67%"}
```

в результате может выглядеть так (требуются скрипты на стороне джиры)
![Jira task]({{ "/resources/images/code_coverage_jira.png" }})
