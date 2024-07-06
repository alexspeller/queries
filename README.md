# Queries

The Query object is designed to encapsulate a complex database query in a single object, allowing building of complex
views on data whilst handling common postgres / activerecord gotchas in a uniform way.

## Basic Idea

A query takes two inputs, a base scope and a params hash, which would could be request query params or from any source.

Executing a query returns an activerecord relation containing records of the type of the scope you sent in.

```rb
result = Query::PickingReport.execute(site.deployment_locations, params)

result.count # 123
result.class # ActiveRecord::Relation
```

However, that result may have some extensions to it (e.g. group-aware counting, built-in pagy, etc), and
also the objects within may be decorated with some extra methods if necessary

## Filtering / Sorting / Pagination

If you want to do things like query via a join table whilst aggregating over a join table, this requires some
careful handling to ensure things work as expected. For example, see sorted_locations_with_aggregate_fields
in PickingController - it's massively painful to say "give me locations that match this inventory object,
but also sum the expected count across the other non-matching inventory objects" because usually those other
objects will be filtered out by the join before aggregation.

Also things like sorting by an aggregate field, paginating, validating input etc are a pain to do in controller methods.

So the idea is to have a DSL that does the correct thing by default, makes it clear you're not using plain active record
objects if you need some decoration, and is simple to use wherever needed.

## Example

```rb
class Query::PickingReport < Query::Application
  default_scope { pick_face } # called on the input scope, in this case we only want pick locations

  # prevent N+1 - and take an opinionated view that preload is the way to do this, not with joins, because
  # the join approach massively complicates queries and is usually less performant for 1:many releationships
  preload localized_inventory_objects: :inventory_object

  # Add an aggregate field (defaults to sum_expected_count) that will correctly sum accross
  # the association and field passed, even if filtering by that field.

  # For example, if you're searching for sku 123, a naive join and group by will only
  # sum accross the associated localized inventory objects matching that sku - but
  # we want to sum accross all associated ones which requires a partition or CTE approach
  sum :localized_inventory_objects, :expected_count
  sum :localized_inventory_objects, :estimated_count

  # Take in a param, with a default value, and apply scopes as needed.

  # This handles array params by default with "OR" - so if you provide `{status: ["limited_discrepancy", "possible_discrepancy"]}`,
  # you'll get results matching both combined with "OR". We could add an "AND" option too if we want to
  param :status, default: "possible_discrepancy" do |status|
    case status
    when "limited_discrepancy"
      picking_ok
    when "possible_discrepancy"
      scope.picking_discrepancy
    when "wms_mismatch"
      error
    when "obstructed"
      obstructed
    when "ignored"
      ignored
    end
  end

  # Add a CTE for data to be referenced elsewhere in the query
  with :latest_deployments, Deployment
    .select("DISTINCT ON (site_id, name) id, name, site_id, stopped_at")
    .order(:site_id, :name, stopped_at: :desc)

  # Filter exact matches with inputs from query params, accross join tables
  # in this case, params `{sku: "123"}` or `{sku: ["123", "456"]}` will return deployment
  # locations matching any passed in sku
  filter :sku, inventory_object: :sku

  # perform ILIKE queries on the named fields, either on the record directly or on
  # fields on assocations, by using the `query` key in params
  text_query :name,
    inventory_objects: %i[sku product_code marker supplier client awb description],
    marker_detections: [:marker]


  # which fields to select
  select(:name)


  # (TODO) order by {pick_sort_order: "name", pick_sort_directio: "desc"}, including handling
  # of e.g. aggregate fields
  order_params :pick_sort_order, :pick_sort_direction

  # (TODO) add ability to get facet information
  facets :name,
    inventory_objects: %i[sku product_code marker supplier client awb description],
    marker_detections: [:marker]


 	# (TODO) decorate returned relation to add extra methods to the relation that will be useful
 	decorate_relation do
 		def special_count
 			where(foo: "special").count
 		end
 	end

 	# (TODO) decorate returned objects so that they have extra methods on them
 	# (this is my idea to avoid having model methods that only work in some cases -
 	#  - put them here so they are explicitly defined in the query and easy to see
 	# where they come from)
 	decorate_results do
 		def discrepency_percentage
 			sum_estimated_count / sum_expected_count * 100
 		end
 	end
end
```