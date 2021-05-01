---
layout: article
tags: Python
title: Gitlab API 
mathjax: true
key: Linux
---

## install
```
pip install python-gitlab
```

[Documentation](https://python-gitlab.readthedocs.io/en/stable/api-usage.html)


## API
```
# 通过UMB里面的stream找到project：
        "stream": "rhel8"

# 创建gitlab对象
gl=gitlab.Gitlab('https://gitlab.com/', private_token='dsfd')
gl.auth()
# 创建工程对象
project = gl.projects.get(project_id)

通过UMB得到mr_id

# 得到merge request对象
mr = project.mergerequests.get(mr_id)

# 获取mr的changes
changes = mr.changes()
[ x.get("old_path") for x in changes.get("changes")]

#### changes是个字典：
print(changes.keys())
dict_keys(['id', 'iid', 'project_id', 'title', 'description', 'state', 'created_at', 'updated_at', 'merged_by', 'merged_at', 'closed_by', 'closed_at', 'target_branch', 'source_branch', 'user_notes_count', 'upvotes', 'downvotes', 'author', 'assignees', 'assignee', 'reviewers', 'source_project_id', 'target_project_id', 'labels', 'work_in_progress', 'milestone', 'merge_when_pipeline_succeeds', 'merge_status', 'sha', 'merge_commit_sha', 'squash_commit_sha', 'discussion_locked', 'should_remove_source_branch', 'force_remove_source_branch', 'allow_collaboration', 'allow_maintainer_to_push', 'reference', 'references', 'web_url', 'time_stats', 'squash', 'task_completion_status', 'has_conflicts', 'blocking_discussions_resolved', 'approvals_before_merge', 'subscribed', 'changes_count', 'latest_build_started_at', 'latest_build_finished_at', 'first_deployed_to_production_at', 'pipeline', 'head_pipeline', 'diff_refs', 'merge_error', 'user', 'changes', 'overflow'])

### changes里面的changes是一个列表，列表中是字典
print(changes.get("changes")[0].keys())
dict_keys(['old_path', 'new_path', 'a_mode', 'b_mode', 'new_file', 'renamed_file', 'deleted_file', 'diff'])
```
