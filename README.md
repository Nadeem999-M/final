Load Raw Data and the Dataset Structure 
load("Workspace_Raw_Data.mat")

Data Cleaning 
Remove Rows and columns from Main_Dataset 
Check for any missing values and remove them.
% Remove rows from 1 to 6 from Main_Dataset and keep the first 3 columns only
for idx = 1:length(dataStruct)
    if isfield(dataStruct(idx), 'Main_Dataset') && ...
       isfield(dataStruct(idx).Main_Dataset, 'data') && ...
       ~isempty(dataStruct(idx).Main_Dataset.data)

        C = dataStruct(idx).Main_Dataset.data;  % Cell array, 1st row = headers

        % Remove rows 1 to 6 and keep only the first 3 columns
        filteredData = C(7:end, 1:8);  % Keep rows from 7 onwards and first 3 columns

        % Convert to table and label the columns
        dataTable = cell2table(filteredData, 'VariableNames', {'Alpha', 'Cl', 'Cd','CDp','Cm','Top Xtr','Bot Xtr','Cpmin'});

        % Store the table back into the struct
        dataStruct(idx).Main_Dataset.data = dataTable;
    end
end

% Check for any missing values and remove them
for idx = 1:length(dataStruct)
    if isfield(dataStruct(idx), 'Main_Dataset') && ...
       isfield(dataStruct(idx).Main_Dataset, 'data') && ...
       ~isempty(dataStruct(idx).Main_Dataset.data)

        dataTable = dataStruct(idx).Main_Dataset.data;  % Table format

        % Remove rows with any missing values
        dataTable = rmmissing(dataTable);

        % Store the cleaned table back into the struct
        dataStruct(idx).Main_Dataset.data = dataTable;
    end
end

Feature Engineering 
Add feature Cl/Cd 
% Calculate Cl/Cd and add to the 4th column for all the files in Main_Dataset
for idx = 1:length(dataStruct)
    if isfield(dataStruct(idx), 'Main_Dataset') && ...
       isfield(dataStruct(idx).Main_Dataset, 'data') && ...
       ~isempty(dataStruct(idx).Main_Dataset.data)

        dataTable = dataStruct(idx).Main_Dataset.data;  % Table format

        % Extract Cl and Cd columns
        Cl = dataTable.Cl;  % Assuming Cl is in the second column
        Cd = dataTable.Cd;  % Assuming Cd is in the third column

        % Calculate Cl/Cd, avoiding division by zero
        ClCd = Cl ./ Cd;
        ClCd(Cd == 0) = NaN;  % Set Cl/Cd to NaN where Cd is zero

        % Add Cl/Cd to the table as a new column
        dataTable.ClCd = ClCd;

        % Store the updated table back into the struct
        dataStruct(idx).Main_Dataset.data = dataTable;
    end
end

Coordinate Geometry Flattening into the MainDataset 
% Flatten Coordinate Geometry into the MainDataset
for i = 1:length(dataStruct)
    entry = dataStruct(i);

    % Skip if missing geometry or main dataset
    if isempty(entry.Coordinate_Geometry.data) || isempty(entry.Main_Dataset.data)
        continue;
    end

    % Geometry: [x y] → flatten into row: [x1, x2, ..., y1, y2, ...]
    geom = entry.Coordinate_Geometry.data;
    x = geom(:, 1)';
    y = geom(:, 2)';
    geom_flat = [x, y];  % 1 row, 2N columns

    % Extract and convert main data
    raw = entry.Main_Dataset.data;
    if iscell(raw)
        % Convert cell to table with headers
        headers = string(raw(1, :));
        dataRows = raw(2:end, :);
        T = cell2table(dataRows, 'VariableNames', headers);
    elseif istable(raw)
        T = raw;
    else
        warning("Unrecognized Main_Dataset format at index %d", i);
        continue;
    end

    % Create geometry table with repeated row to match T
    nRows = height(T);
    geomTable = array2table(repmat(geom_flat, nRows, 1));

    % Create variable names: x1, x2, ..., y1, y2, ...
    nPoints = length(x);
    xVars = "x" + (1:nPoints);
    yVars = "y" + (1:nPoints);
    geomTable.Properties.VariableNames = [xVars, yVars];

    % Insert geometry columns after 4th column (Cl/Cd)
    T = [T(:,1:9), geomTable, T(:,10:end)];

    % Update struct
    dataStruct(i).Main_Dataset.data = T;
end


EDA
BiPlot
Correlation Matrix
Pareto Chart
Scatter Plot 
figure;
hold on;
for idx = 1:length(dataStruct)
    if isempty(dataStruct(idx).Main_Dataset.data)
        T = dataStruct(idx).Main_Dataset.data;
        T = T(:,1:9); % Only columns 1–9
        numericVars = varfun(@isnumeric, T, 'OutputFormat', 'uniform');
        X = T{:, numericVars};
        if size(X,2) > 1 % PCA requires at least 2 variables
            [, score] = pca(X, 'Rows', 'complete');
            scatter(score(:,1), score(:,2), 'DisplayName', ['Dataset ' num2str(idx)]);
        end
    end
end
title('BiPlot: All Datasets (Columns 1 to 9)');
xlabel('PC1');
ylabel('PC2');
legend show;


corr plot
% Define variable names for columns 1 to 9
varNames = {'Alpha', 'Cl', 'Cd', 'CDp', 'Cm', 'Top Xtr', 'Bot Xtr', 'Cpmin', 'ClCd'};

% Aggregate numeric data from all datasets (columns 1 to 9)
allData = [];
for idx = 1:length(dataStruct)
    if ~isempty(dataStruct(idx).Main_Dataset.data)
        T = dataStruct(idx).Main_Dataset.data;
        T = T(:,1:9); % Restrict to columns 1-9
        numericVars = varfun(@isnumeric, T, 'OutputFormat', 'uniform');
        allData = [allData; T{:, numericVars}];
    end
end

% Compute correlation matrix
R = corr(allData, 'Rows', 'complete');

% Create heatmap with custom axis labels
figure;
h = heatmap(varNames, varNames, R);

% Add title and axis labels
h.Title = 'Combined Correlation Matrix (Columns 1 to 9)';
h.XLabel = 'Variables';
h.YLabel = 'Variables';

Pareto
% % varNames = {'Alpha', 'Cl', 'Cd', 'CDp', 'Cm', 'Top Xtr', 'Bot Xtr', 'Cpmin', 'ClCd'};
% % 
% % for idx = 1:length(dataStruct)
% %     if ~isempty(dataStruct(idx).Main_Dataset.data)
% %         T = dataStruct(idx).Main_Dataset.data;
% %         T = T(:,1:9);  % Use columns 1 to 9
% % 
% %         figure;
% %         % Create Pareto chart for Cl (2nd column) as example
% %         [bars, line] = pareto(T.Cl);
% % 
% %         % Customize X-axis tick labels
% %         ax = gca;
% %         ax.XTick = 1:length(T.Cl);  % Set tick locations
% %         % For large datasets, you may want to limit tick labels or rotate them
% %         % Here, we label bars with variable name 'Cl' repeated (or customize as needed)
% %         % Since Pareto bars correspond to data points, not variables, 
% %         % we can label bars with row indices or skip labels for clarity.
% % 
% %         % Add title and axis labels
% %         title(['Pareto Chart of Cl - Dataset ' num2str(idx)]);
% %         xlabel('Data Point Index');
% %         ylabel('Cl Value');
% % 
% %         % Optional: rotate x-axis labels if you add custom labels
% %         % ax.XTickLabelRotation = 45;
% %     end
% % end

3D scatter
varNames = {'Alpha', 'Cl', 'Cd', 'CDp', 'Cm', 'Top Xtr', 'Bot Xtr', 'Cpmin', 'ClCd'};

figure(1);
hold on;
for idx = 1:length(dataStruct)
    if ~isempty(dataStruct(idx).Main_Dataset.data)
        T = dataStruct(idx).Main_Dataset.data;
        T = T(:,1:9);  % Use columns 1 to 9
        
        scatter3(T.Alpha, T.Cl, T.Cd, 36, T.ClCd, 'filled', ...
            'DisplayName', ['Dataset ' num2str(idx)]);
    end
end
xlabel('Alpha');
ylabel('Cl');
zlabel('Cd');
title('3D Scatter Plot: Alpha vs Cl vs Cd (Columns 1 to 9)');
colorbar;
% legend show;
grid on;

Plots 
Cl vs Cd, 
Cl vs Alpha, 
Cd vs Alpha, 
Cl & Cd vs Alpha (Superimposed Graph)
Geometry (X vs Y)
% Call the airfoilPlotterGUI function with the cleaned and processed dataStruct
airfoilPlotterGUI(dataStruct)
