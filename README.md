/**** Start of imports. If edited, may not auto-convert in the playground. ****/
var Tehran = ee.FeatureCollection("projects/ee-soheilasangi/assets/Tehran"),
    o3 = ee.ImageCollection("COPERNICUS/S5P/OFFL/L3_O3_TCL");
/***** End of imports. If edited, may not auto-convert in the playground. *****/
var studyarea = Tehran;

Map.addLayer(studyarea, {}, 'Study Area', true, 0.5);
Map.centerObject(studyarea);

var start = ee.Date('2024-05-01');
var end   = ee.Date('2024-06-01');
var scale = 100000;

function dailyMean(images, start, end) {
  var n = end.difference(start, 'day').round();
  var idx = ee.List.sequence(0, n.subtract(1));
  return ee.ImageCollection(idx.map(function(i) {
    i = ee.Number(i);
    var from = start.advance(i, 'day');
    var to = from.advance(1, 'day');
    var img = images.filterDate(from, to).mean();
    return img.set('system:time_start', from.millis())
              .set('date', from.format('yyyy-MM-dd'));
  }));
}

function plot(collection, band, title, viz) {
  var c2 = dailyMean(collection, start, end);
  print('Daily size ' + title + ':', c2.size());
  print('Daily dates ' + title + ':', c2.aggregate_array('date'));
  var chart = ui.Chart.image.series({
    imageCollection: c2.select(band),
    region: studyarea,
    reducer: ee.Reducer.mean(),
    scale: scale,
    xProperty: 'system:time_start'
  }).setChartType('LineChart').setOptions({
    title: title,
    hAxis: {title: 'Date'},
    vAxis: {title: band}
  });
  print(chart);
  Map.addLayer(c2.first().select(band), viz, title + ' first day');
}

// Try tropospheric O3 (only 20S–20N)
var tcl = ee.ImageCollection('COPERNICUS/S5P/OFFL/L3_O3_TCL')
  .filterBounds(studyarea)
  .filterDate(start, end)
  .map(function(img){
    return img.updateMask(img.select('qa_value').gte(70)).clip(studyarea);
  });

tcl.size().evaluate(function(n){
  if (n === 0) {
    print('No O3_TCL coverage here (outside 20S–20N). Using total column O3.');
    var o3 = ee.ImageCollection('COPERNICUS/S5P/OFFL/L3_O3')
      .filterBounds(studyarea)
      .filterDate(start, end)
      .map(function(img){
        return img.updateMask(img.select('cloud_fraction').lte(0.5)).clip(studyarea);
      });
    plot(
      o3,
      'O3_column_number_density',
      'Total Column O3 (mol/m^2)',
      {min: 0.12, max: 0.15, palette: ['black','blue','purple','cyan','green','yellow','red']}
    );
  } else {
    plot(
      tcl,
      'ozone_tropospheric_vertical_column',
      'Tropospheric O3 (mol/m^2)',
      {min: 0.0, max: 0.02, palette: ['black','blue','purple','cyan','green','yellow','red']}
    );
  }
});
  
