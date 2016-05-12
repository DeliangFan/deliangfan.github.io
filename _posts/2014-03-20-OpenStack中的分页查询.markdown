---
layout: post
title:  "OpenStack 的分页查询"
categories: OpenStack
---


Heat 在 Havana 版本首次成为正式模块，为 OpenStack 提供了令人兴奋的资源编排功能，但是和成熟项目相比，它缺失了某些小特性，比如不支持分页查询。随着用户的 instance、volume 和 stack 等资源的数量越来越多，分页查询的功能越发显得重要，而且在很多其它的场景，分页查询的功能也非常实用。本文以 Nova 为例，介绍 OpenStack 是如何实现分页查询的功能，用于学习和借鉴。

以查询 [list instance](http://developer.openstack.org/api-ref-compute-v2.1.html#listServers) 为例，注意如下两个 Request parameters：

~~~
+-------------------+-------+---------+--------------------------------------------+
| Parameter         | Style | Type    | Description                                |
+-------------------+-------+---------+--------------------------------------------+
| limit (Optional)  | query | xsd:int | Requests a page size of items. Returns a   |
|                   |       |         | number of items up to a limit value. Use   |
|                   |       |         | the limit parameter to make an initial     |
|                   |       |         | limited request and use the ID of the      |
|                   |       |         | last-seen item from the response as the    |
|                   |       |         | marker parameter value in a subsequent     |
|                   |       |         | limited request.                           |
+-------------------+-------+---------+--------------------------------------------+
| marker (Optional) | query | xsd:int | The ID of the last-seen item. Use the      |
|                   |       |         | limit parameter to make an initial limited |
|                   |       |         | request and use the ID of the last-seen    |
|                   |       |         | item from the response as the marker       |
|                   |       |         | parameter value in a subsequent limited    |
|                   |       |         | request.                                   |
+-------------------+-------+---------+--------------------------------------------+
~~~

通俗的说，limit 表示返回的 instance 数量，marker 表示起始的 instance id(查询结果不包含该 instance)，用法如下：

~~~
/v2.1/​{tenant_id}​/servers?limit=10&marker=a291599e-6de2-41a6-88df-c443ddcef70d
~~~

接下以 havana 版本的 nova 为例介绍分页查询的原理：

nova/api/openstack/common.py 中如下函数定义从 http url 中获取 limit 和 marker 的函数：

~~~ python
def get_limit_and_marker(request, max_limit=CONF.osapi_max_limit):
    """get limited parameter from request."""
    params = get_pagination_params(request)
    limit = params.get('limit', max_limit)
    limit = min(max_limit, limit)
    marker = params.get('marker')

    return limit, marker


def limited_by_marker(items, request, max_limit=CONF.osapi_max_limit):
    """Return a slice of items according to the requested marker and limit."""
    limit, marker = get_limit_and_marker(request, max_limit)

    limit = min(max_limit, limit)
    start_index = 0
    if marker:
        start_index = -1
        for i, item in enumerate(items):
            if 'flavorid' in item:
                if item['flavorid'] == marker:
                    start_index = i + 1
                    break
            elif item['id'] == marker or item.get('uuid') == marker:
                start_index = i + 1
                break
        if start_index < 0:
            msg = _('marker [%s] not found') % marker
            raise webob.exc.HTTPBadRequest(explanation=msg)
    range_end = start_index + limit
    return items[start_index:range_end]
~~~

nova/api/openstack/compute/servers.py 中的 _get_servers 函数调用了以上函数，获取 limit 和 marker 参数，并传入所调用的查询函数：

~~~ python
def _get_servers(self, req, is_detail):
    ......
    limit, marker = common.get_limit_and_marker(req)
    try:
        instance_list = self.compute_api.get_all(context,
                                                 search_opts=search_opts,
                                                 limit=limit,
                                                 marker=marker,
                                                 want_objects=True)
    ......
~~~

self.compute_api.get_all 最终调用 nova/db/sqlalchemy/api.py 进行分页查询：

~~~ python
@require_context
def instance_get_all_by_filters(context, filters, sort_key, sort_dir,
                                limit=None, marker=None, columns_to_join=None):
......
    # paginate query
    if marker is not None:
        try:
            marker = _instance_get_by_uuid(context, marker, session=session)
        except exception.InstanceNotFound:
            raise exception.MarkerNotFound(marker)
    query_prefix = sqlalchemyutils.paginate_query(query_prefix,
                           models.Instance, limit,
                           [sort_key, 'created_at', 'id'],
                           marker=marker,
                           sort_dir=sort_dir)
......
~~~

而 sqlalchemyutils 又是调用 nova/openstack/common/db/sqlalchemy/utils.py 的 paginate_query 实现分页查询：

~~~ python
# copy from glance/db/sqlalchemy/api.py
def paginate_query(query, model, limit, sort_keys, marker=None,
                   sort_dir=None, sort_dirs=None):
    """Returns a query with sorting / pagination criteria added.

    Pagination works by requiring a unique sort_key, specified by sort_keys.
    (If sort_keys is not unique, then we risk looping through values.)
    We use the last row in the previous page as the 'marker' for pagination.
    So we must return values that follow the passed marker in the order.
    With a single-valued sort_key, this would be easy: sort_key > X.
    With a compound-values sort_key, (k1, k2, k3) we must do this to repeat
    the lexicographical ordering:
    (k1 > X1) or (k1 == X1 && k2 > X2) or (k1 == X1 && k2 == X2 && k3 > X3)

    We also have to cope with different sort_directions.

    Typically, the id of the last row is used as the client-facing pagination
    marker, then the actual marker object must be fetched from the db and
    passed in to us as marker.

    :param query: the query object to which we should add paging/sorting
    :param model: the ORM model class
    :param limit: maximum number of items to return
    :param sort_keys: array of attributes by which results should be sorted
    :param marker: the last item of the previous page; we returns the next
                    results after this value.
    :param sort_dir: direction in which results should be sorted (asc, desc)
    :param sort_dirs: per-column array of sort_dirs, corresponding to sort_keys

    :rtype: sqlalchemy.orm.query.Query
    :return: The query with sorting/pagination added.
    """

    if 'id' not in sort_keys:
        # TODO(justinsb): If this ever gives a false-positive, check
        # the actual primary key, rather than assuming its id
        LOG.warn(_('Id not in sort_keys; is sort_keys unique?'))

    assert(not (sort_dir and sort_dirs))

    # Default the sort direction to ascending
    if sort_dirs is None and sort_dir is None:
        sort_dir = 'asc'

    # Ensure a per-column sort direction
    if sort_dirs is None:
        sort_dirs = [sort_dir for _sort_key in sort_keys]

    assert(len(sort_dirs) == len(sort_keys))

    # Add sorting
    for current_sort_key, current_sort_dir in zip(sort_keys, sort_dirs):
        sort_dir_func = {
            'asc': sqlalchemy.asc,
            'desc': sqlalchemy.desc,
        }[current_sort_dir]

        try:
            sort_key_attr = getattr(model, current_sort_key)
        except AttributeError:
            raise InvalidSortKey()
        query = query.order_by(sort_dir_func(sort_key_attr))

    # Add pagination
    if marker is not None:
        marker_values = []
        for sort_key in sort_keys:
            v = getattr(marker, sort_key)
            marker_values.append(v)

        # Build up an array of sort criteria as in the docstring
        criteria_list = []
        for i in range(0, len(sort_keys)):
            crit_attrs = []
            for j in range(0, i):
                model_attr = getattr(model, sort_keys[j])
                crit_attrs.append((model_attr == marker_values[j]))

            model_attr = getattr(model, sort_keys[i])
            if sort_dirs[i] == 'desc':
                crit_attrs.append((model_attr < marker_values[i]))
            elif sort_dirs[i] == 'asc':
                crit_attrs.append((model_attr > marker_values[i]))
            else:
                raise ValueError(_("Unknown sort direction, "
                                   "must be 'desc' or 'asc'"))

            criteria = sqlalchemy.sql.and_(*crit_attrs)
            criteria_list.append(criteria)

        f = sqlalchemy.sql.or_(*criteria_list)
        query = query.filter(f)

    if limit is not None:
        query = query.limit(limit)

    return query
~~~